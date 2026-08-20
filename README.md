
COMPLETE GUIDE: SOLVING SILENT BITLOCKER ENCRYPTION ISSUES IN INTUNE
--------------------------------------------------------------------------------

Document Overview:
This comprehensive guide provides step-by-step instructions to resolve issues where 
Silent BitLocker fails to enable after deleting and re-enrolling a Windows device in 
Microsoft Intune.

--------------------------------------------------------------------------------
SECTION 1: BASIC CLEANUP & REGISTRY CACHE RESET
--------------------------------------------------------------------------------
When a device is re-enrolled, local registry keys and cached scheduled tasks from the
previous Intune enrollment often prevent new encryption policies from triggering.

Step 1.1: Clear Local Policy Manager Registry Entries
Open PowerShell as Administrator on the target machine and execute:

  Remove-Item -Path "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker" -Recurse -Force -ErrorAction SilentlyContinue
  Remove-Item -Path "HKLM:\SOFTWARE\Microsoft\PolicyManager\providers\*\default\Device\BitLocker" -Recurse -Force -ErrorAction SilentlyContinue

Step 1.2: Reset BitLocker Scheduled Tasks
Run the following PowerShell command:

  Get-ScheduledTask -TaskPath "\Microsoft\Windows\BitLocker\" | Enable-ScheduledTask

--------------------------------------------------------------------------------
SECTION 2: MDM POLICY SYNC & RE-EVALUATION
--------------------------------------------------------------------------------
Force the device to communicate immediately with Microsoft Intune to fetch updated policies.

Step 2.1: GUI Sync Method
1. Open Windows Settings (Win + I).
2. Go to Accounts > Access work or school.
3. Select your Work/School account -> Click Info.
4. Scroll down and click "Sync".

Step 2.2: PowerShell Force Sync Method (Run as Admin)
  Get-ScheduledTask | Where-Object {$_.TaskPath -like "*Microsoft\Windows\EnterpriseMgmt*"} | Start-ScheduledTask

--------------------------------------------------------------------------------
SECTION 3: INTUNE MANAGEMENT EXTENSION (IME) REPAIR
--------------------------------------------------------------------------------
If policy sync is non-responsive, repairing the Intune Management Extension agent 
forces a full re-evaluation of MDM scripts and configurations.

Step 3.1: Restart Intune Service
1. Open Services (Win + R -> services.msc).
2. Locate "Microsoft Intune Management Extension".
3. Right-click and select "Restart".

Step 3.2: Reinstall IME Agent
1. Go to Settings > Apps > Installed apps.
2. Search for "Microsoft Intune Management Extension" and uninstall it.
3. Delete cached files in: C:\ProgramData\Microsoft\IntuneManagementExtension
4. Perform an MDM Sync (from Section 2.1). Intune will automatically reinstall IME.

--------------------------------------------------------------------------------
SECTION 4: ADVANCED REGISTRY & TASK SCHEDULER CLEANUP
--------------------------------------------------------------------------------
Stale Enrollment GUIDs can cause duplicate enrollment conflicts.

Step 4.1: Clean Old Enrollment GUIDs
1. Press Win + R, type regedit, and press Enter.
2. Navigate to: HKLM\SOFTWARE\Microsoft\Enrollments
3. Look through the GUID subfolders (e.g., {FA0081XX-...}).
4. Keep the active GUID folder containing "manage.microsoft.com" under DiscoveryServiceFullURL.
5. Delete any duplicate/stale GUID folders corresponding to previous enrollments.

Step 4.2: Clean Task Scheduler
1. Press Win + R, type taskschd.msc, and press Enter.
2. Navigate to Task Scheduler Library > Microsoft > Windows > EnterpriseMgmt.
3. Delete any orphaned or disabled tasks sitting under old GUID folders.

--------------------------------------------------------------------------------
SECTION 5: SYSTEM PREREQUISITES & HARDWARE CHECKS
--------------------------------------------------------------------------------
Silent BitLocker will silently fail if hardware or policy requirements are not met.

Step 5.1: Check TPM (Trusted Platform Module) Status
1. Press Win + R, type tpm.msc, and press Enter.
2. Confirm the Status displays "The TPM is ready for use".
3. If old keys exist, select "Clear TPM" (Requires a machine reboot).

Step 5.2: Check Drive Health & Volume Status
Run PowerShell as Admin:

  Get-BitLockerVolume

Ensure ProtectionStatus is "Off" and VolumeStatus is "FullyDecrypted". If stuck,
resolve disk issues first.

Step 5.3: Verify Intune Policy Settings
Ensure the following settings are configured in Intune Endpoint Security Policy:
- Require Device Encryption: Enabled
- Allow Warning for Other Disk Encryption: Block
- Allow Standard Users to Enable Encryption during Azure AD Join: Yes
- Require TPM: Required

--------------------------------------------------------------------------------
SECTION 6: MANUAL FORCE-TRIGGER (IF SILENT ENCRYPTION STILL DOESN'T START)
--------------------------------------------------------------------------------
If all configurations are verified but encryption doesn't start automatically, 
you can manually trigger Silent Encryption using PowerShell (Admin):

  $BLV = Get-BitLockerVolume -MountPoint "C:"
  if ($BLV.ProtectionStatus -eq "Off") {
      Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes256 -UsedSpaceOnly -SkipHardwareTest
  }

*Note: Once initiated, BitLocker will encrypt the drive and automatically sync and 
store the Recovery Key in Microsoft Entra ID / Intune.

--------------------------------------------------------------------------------
SECTION 7: TROUBLESHOOTING VIA EVENT VIEWER
--------------------------------------------------------------------------------
To pinpoint exact errors causing failures, check system logs:

1. Press Win + R, type eventvwr.msc, and press Enter.
2. Go to: Applications and Services Logs > Microsoft > Windows > BitLocker-API > Management.
3. Review any Warning or Error logs. Common issues include:
   - TPM not ready or cleared.
   - User profile lacking encryption permissions.
   - Unallocated disk space preventing full volume protection.
---------------------------------------------------------------------------------

