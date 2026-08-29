# Fix Slow Windows Explorer Folder Loading

If Windows Explorer suddenly takes an unusually long time to open folders such as **Downloads**, try the following:

1. **Repair the Windows component store — Command Prompt / Terminal (Admin)**
   ```cmd
   DISM.exe /Online /Cleanup-Image /RestoreHealth
   ```

2. **Repair Windows system files — Command Prompt / Terminal (Admin)**
   ```cmd
   sfc /scannow
   ```

3. **Check the filesystem — Command Prompt / Terminal (Admin)**
   ```cmd
   chkdsk C: /scan
   ```

4. **Check disk health — PowerShell**
   ```powershell
   Get-PhysicalDisk | Select FriendlyName, HealthStatus, OperationalStatus
   ```
   Requires the Windows `Storage` module. Available in Windows PowerShell 5.1 and generally usable from PowerShell 7 on Windows.

5. **Restart Windows.**

If `chkdsk` reports no filesystem errors and the disks report healthy, the **DISM → SFC → restart** sequence may still resolve Explorer folder-loading delays caused by damaged Windows components or system files.
