# Windows Explorer Hangs When Selecting Certain WAV Files

*A troubleshooting guide based on an actual Explorer hang that got much more involved than it deserved.*

## What this covers

Use this when **File Explorer hangs just from selecting, right-clicking, previewing, or opening one particular `.wav`**, while other WAVs work and tools such as FFmpeg can read the problem file normally.

Explorer may inspect a WAV through thumbnail, property, Media Foundation, search/indexing, shell-extension, or third-party code before you ever "open" it.

The real incident behind this guide ended like this:

1. become mildly drunk;
2. spend far too long debugging Explorer;
3. update Windows;
4. run DISM, SFC and CHKDSK;
5. forget about the problem;
6. come back weeks later;
7. discover the **same original WAVs now work**.

So start with repair. If that does not work, continue into ProcDump, WinDbg and ProcMon.

> **Registry rule:** export before changing, test one thing at a time, restore it if it makes no difference.

---

## 0. Become mildly drunk

No reason, this was just annoying to deal with.

---

## 1. Update and repair Windows

Fully update Windows, reboot, then run an elevated shell:

```powershell
DISM.exe /Online /Cleanup-Image /RestoreHealth
sfc /scannow
chkdsk C: /scan
```

Reboot again.

In this case, the untouched problem WAVs eventually began working after this maintenance. The exact repaired component was never proven. Windows Update and DISM/SFC are more plausible than CHKDSK because the filesystem had already appeared healthy.

If the WAV now works: **stop. You won.**

---

## 2. Prove Explorer is the weird part

Test the file without Explorer:

```powershell
ffprobe "C:\path\to\problem.wav"
ffmpeg -i "C:\path\to\problem.wav" -f null NUL
Get-Item "C:\path\to\problem.wav"
Get-FileHash "C:\path\to\problem.wav"
```

If these work while Explorer hangs, basic filesystem access and independent WAV parsing are functioning. Suspect Windows' file-type-specific inspection path.

---

## 3. Isolate the file

Put one offender in an empty directory with a boring name:

```powershell
New-Item -ItemType Directory -Path "C:\wav-test" -Force
Copy-Item "C:\path\to\problem.wav" "C:\wav-test\test.wav"
```

Open only `C:\wav-test` and single-click `test.wav`.

This removes filename weirdness, neighboring files, original folder state, and unrelated thumbnail work.

---

## 4. Rebuild the WAV

Strip normal metadata and create a fresh PCM WAV:

```powershell
ffmpeg -i "C:\wav-test\test.wav" `
  -map_metadata -1 `
  -c:a pcm_s24le `
  "C:\wav-test\clean.wav"
```

Also make a deliberately boring 16-bit / 48 kHz stereo version:

```powershell
ffmpeg -i "C:\wav-test\test.wav" `
  -map_metadata -1 `
  -ar 48000 `
  -ac 2 `
  -c:a pcm_s16le `
  "C:\wav-test\test-16.wav"
```

Interpretation:

- rebuilt WAV works -> suspect original RIFF/container/metadata details;
- rebuilt WAV still hangs -> ordinary metadata, high bit depth and high sample rate are weaker suspects.

In this case, **both fresh WAVs still hung Explorer**.

---

## 5. Do the `.wav` vs `.bin` A/B test

Make a byte-identical copy with a neutral extension:

```powershell
Copy-Item "C:\wav-test\test-16.wav" "C:\wav-test\test.bin"
```

If this happens:

```text
same bytes + .wav -> Explorer hangs
same bytes + .bin -> Explorer works
```

then the bytes alone are not sufficient. **Windows recognizing the file as WAV is required for the trigger.**

That strongly implicates WAV-specific parsing/inspection rather than NTFS, the directory, or simple reads.

This was the strongest discriminator in the original case: `.wav` killed Explorer; the byte-identical `.bin` was fine.

---

## 6. Disable obvious Explorer inspection UI

Temporarily turn off:

- Preview pane;
- Details pane;
- `Folder Options -> View -> Always show icons, never thumbnails`.

Restart Explorer:

```powershell
Stop-Process -Name explorer -Force
```

Retest. A failure here does **not** completely acquit thumbnail/property systems; Explorer can still request icons and properties for selected items.

---

## 7. Confirm it is a hang

Reliability Monitor may show:

```text
Problem Event Name: AppHangB1
Application Name: explorer.exe
```

Record the OS build, Explorer version, timestamp and hang signature. A hang often has no useful "faulting module" because no exception occurred; a thread simply stopped making progress.

---

# Advanced diagnostics

If the WAV still hangs after the basic tests, congratulations: you are now debugging Explorer.

## 8. Capture Explorer with ProcDump

Download **Sysinternals ProcDump**.

```powershell
New-Item -ItemType Directory -Path "C:\ExplorerDumps" -Force
Get-Process explorer | Select-Object Id, MainWindowTitle, StartTime
```

Identify the Explorer PID hosting the affected window, then arm the hung-window trigger:

```powershell
procdump64.exe -accepteula -h <PID> "C:\ExplorerDumps"
```

Select the WAV and reproduce the hang. ProcDump should write a `.dmp` into `C:\ExplorerDumps`.

---

## 9. Inspect the dump with WinDbg

Open the dump in **WinDbg x64**:

```text
.symfix
.reload
!analyze -hang
```

If the headline is merely something like:

```text
win32u!NtUserWaitMessage
```

that may only mean a GUI thread is waiting. Do not immediately blame `win32u.dll`.

Dump all thread stacks:

```text
~* kb
```

Useful extras:

```text
!runaway 7
lm
~<thread-number>s
kb
lmvm <module-name>
```

Interesting names include:

```text
thumbcache!CThumbnailCache::_PerformFullExtractionCore
windows_storage!CThumbnailExtractTask
shell32!CVerbStateTask
combase!CSyncClientCall
propsys!
mfplat!
mfsrcsnk!
rpcrt4!
```

Third-party DLLs matter when they appear in the blocked path, not merely because they are loaded.

This case showed shell/thumbnail/COM activity here, but no conclusive culprit.

---

## 10. Use ProcMon to trace what touches the WAV

When WinDbg gives you 1,600 lines of call stacks and no confession, use **Sysinternals Process Monitor**.

Run ProcMon as Administrator.

### Filter the exact WAV

Press `Ctrl+L` and add:

```text
Path | is | C:\wav-test\test.wav | Include
```

Do **not** initially restrict to `explorer.exe`. Helpers may include `dllhost.exe`, search/indexing processes, antivirus/EDR, codec software, or third-party shell components.

Then:

```text
Ctrl+X  -> clear current events
Ctrl+E  -> start capture
```

Single-click the WAV, let the hang reproduce for a few seconds, then:

```text
Ctrl+E  -> stop capture
```

### Inspect the trace

Look at the last and/or repeated accesses to the WAV:

- Process Name
- PID
- Operation
- Result
- Detail

Common operations:

```text
CreateFile
ReadFile
QueryInformationFile
QueryEAFile
CloseFile
```

Do not assume the chronologically final event caused the hang. Look for repeated access or a process that clearly starts inspecting the file when selection occurs.

Double-click an interesting event -> **Stack**.

Look for modules such as:

```text
shell32.dll
thumbcache.dll
propsys.dll
mfsrcsnk.dll
mfplat.dll
codec DLLs
security/EDR DLLs
third-party shell extensions
```

Save the trace as `.PML` if you want to preserve stacks and metadata.

At this point, stop guessing which application is responsible and identify **which component actually touched the file**.

---

# Registry locations to inspect

## 11. `.wav` association

```powershell
reg query "HKCR\.wav" /s
```

The default may be a ProgID such as `VLC.wav` or `WMP11.AssocFile.WAV`. That only tells you the normal file association; it does **not** prove the player causes a selection-time hang.

## 12. WAV ShellEx handlers

```powershell
reg query "HKCR\.wav\ShellEx" /s
```

Important shell registration subkeys include:

```text
{BB2E617C-0920-11D1-9A0B-00C04FC2D6C1}
{E357FCCD-A995-4576-B01F-234630154E96}
```

On the affected system, both pointed to:

```text
{9DBD2C50-62AD-11D0-B806-00C04FD706EC}
```

These were interesting because the dump contained thumbnail extraction activity. Temporarily removing them did **not** fix this case.

Also inspect audio-wide shell behavior:

```powershell
reg query "HKCR\SystemFileAssociations\.wav" /s
reg query "HKCR\SystemFileAssociations\audio\ShellEx" /s
```

## 13. WAV property-handler registrations

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PropertySystem\PropertyHandlers\.wav" /ve
reg query "HKLM\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\PropertySystem\PropertyHandlers\.wav" /ve
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PropertySystem\SystemPropertyHandlers\.wav" /ve
```

A commonly encountered WAV property-handler CLSID is:

```text
{e46787a1-4629-4423-a693-be1f003b2742}
```

It has been implicated in similar Explorer/WAV stalls, but disabling it did **not** solve this case.

---

# Registry A/B tests

## 14. Test the WAV property handler

Back up keys that actually exist:

```powershell
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PropertySystem\PropertyHandlers\.wav" "C:\wav-propertyhandler-64.reg" /y
reg export "HKLM\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\PropertySystem\PropertyHandlers\.wav" "C:\wav-propertyhandler-32.reg" /y
```

If the value is:

```text
{e46787a1-4629-4423-a693-be1f003b2742}
```

a known diagnostic workaround is to invalidate it temporarily:

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PropertySystem\PropertyHandlers\.wav" /ve /t REG_SZ /d "{#e46787a1-4629-4423-a693-be1f003b2742}" /f
```

Apply the equivalent Wow6432Node change only if that key exists. Reboot/log off, test, then restore:

```powershell
reg import "C:\wav-propertyhandler-64.reg"
reg import "C:\wav-propertyhandler-32.reg"
```

If it changes nothing, restore it. **Do not accumulate mystery registry surgery.**

## 15. Test WAV image/thumbnail ShellEx registrations

Back up:

```powershell
reg export "HKCR\.wav\ShellEx" "C:\wav-shellex-backup.reg" /y
```

If present, test handlers individually:

```powershell
reg delete "HKCR\.wav\ShellEx\{BB2E617C-0920-11D1-9A0B-00C04FC2D6C1}" /f
reg delete "HKCR\.wav\ShellEx\{E357FCCD-A995-4576-B01F-234630154E96}" /f
```

Reboot/log off for a clean association refresh. Test, then restore:

```powershell
reg import "C:\wav-shellex-backup.reg"
```

Again: this did **not** fix this case.

---

## 16. Fast interpretation table

| Observation | Strong implication |
|---|---|
| Only certain WAVs hang | File-specific input triggers a WAV path |
| FFmpeg reads them | Independent parsing/basic validity is fine |
| Fresh FFmpeg WAV still hangs | Original metadata/container is less likely |
| 16-bit/48 kHz still hangs | High bit depth/sample rate is less likely |
| `.wav` hangs, byte-identical `.bin` works | WAV recognition/handler path is required |
| Property handler disabled, still hangs | That handler alone does not explain it |
| ShellEx handlers disabled, still hangs | Those handlers alone do not explain it |
| Windows repair later fixes the originals | Broken/outdated/mismatched Windows component becomes the strongest practical explanation |

---

## 17. Practical escape hatch

If you just need the audio, transcode it to another real format and move on.

```powershell
# FLAC
ffmpeg -i "input.wav" -map_metadata -1 -c:a flac "output.flac"

# OGG Vorbis
ffmpeg -i "input.wav" -map_metadata -1 -c:a libvorbis -q:a 5 "output.ogg"

# MP3
ffmpeg -i "input.wav" -map_metadata -1 -c:a libmp3lame -q:a 2 "output.mp3"
```

Renaming `.wav` to `.bin` is a diagnostic control, not a production conversion.

---

# The actual lore

What actually happened in this case:

- specific Sonniss/344 Audio WAVs killed Explorer on selection;
- other WAVs worked;
- FFmpeg read the files normally;
- metadata stripping failed to fix them;
- fresh WAV reconstruction failed;
- 16-bit / 48 kHz conversion failed;
- isolated folder/boring filename failed;
- preview/details/thumbnail toggles failed;
- Reliability Monitor showed `AppHangB1`;
- ProcDump captured a hung `explorer.exe`;
- WinDbg showed shell/thumbnail/COM activity but no clean culprit;
- disabling the WAV property handler failed;
- temporarily removing WAV ShellEx thumbnail/image handlers failed;
- the exact same bytes renamed `.bin` worked, proving `.wav` recognition mattered;
- ProcMon tracing was the next escalation;
- instead, Windows was updated and DISM/SFC/CHKDSK were run;
- the investigation was abandoned;
- weeks later the **original untouched WAVs worked normally**.

So, yes: get mildly drunk, update Windows, repair Windows, forget about it, and discover weeks later that it fixed itself. That was the actual resolution.

The exact root cause therefore remains unproven.

The strongest defensible conclusion is:

> Certain valid WAVs exercised a WAV-specific Windows shell/media path that was broken, outdated, mismatched, or corrupted on that installation. Normal Windows servicing repaired the condition.

Do **not** rewrite that as "Sonniss WAVs are broken," "VLC did it," or "the thumbnail handler did it." None of those were established.

The operational lesson is much simpler:

> **Update Windows. Repair Windows. If that fails, start tracing.**

---

# References

- Microsoft Sysinternals — Process Monitor: https://learn.microsoft.com/en-us/sysinternals/downloads/procmon
- Microsoft Sysinternals — ProcDump: https://learn.microsoft.com/en-us/sysinternals/downloads/procdump
- Microsoft — Shell Extension Handlers: https://learn.microsoft.com/en-us/windows/win32/shell/handlers
- Microsoft — Property Handler Registration: https://learn.microsoft.com/en-us/windows/win32/properties/prophand-reg-dist
- Microsoft — System File Checker: https://support.microsoft.com/en-us/windows/experience/backup-recovery/using-system-file-checker-in-windows
- Microsoft — Repair a Windows Image with DISM: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image
- Microsoft — CHKDSK: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk
- Image-Line — WAV/Explorer property-handler workaround: https://support.image-line.com/action/knowledgebase/?ans=626
