# Windows-Security-Boot-Issue
Secure Boot Fix: Microsoft 2026 Certificate Expiry Rollout


TL;DR: If Secure Boot is blocking your Windows Boot Manager after a recent Windows Update in June 2026, the fix is already on your machine. No downloads, no technician, no BIOS flash required.




The Error You're Seeing

"Windows Boot Manager has been blocked by the current security policy."
"Default Boot Device Missing or Boot Failed."

Everything works fine with Secure Boot disabled. Sound familiar? Keep reading.


What Happened

Microsoft's June 2026 Secure Boot certificate expiry rollout pushed a db (allowed signature database) update to devices. On some firmware implementations — including at least the Lenovo IdeaPad Gaming 3 15IHU6 (BIOS H4CN37WW V2.06) — the firmware overwrote the db instead of appending to it.

This removed the Microsoft Windows Production PCA 2011 certificate from the db. Since bootmgfw.efi (Windows Boot Manager) is signed with that cert, firmware no longer trusts it. Boot fails.

This is a documented firmware bug in Microsoft's official Secure Boot Troubleshooting Guide.


Confirm This Is Your Issue

Run these in admin PowerShell:

powershell# Check bootloader certificate
$sig = Get-AuthenticodeSignature "S:\EFI\Microsoft\Boot\bootmgfw.efi"
$sig.Status
$sig.SignerCertificate.Issuer

Expected output if affected:

Valid
CN=Microsoft Windows Production PCA 2011, O=Microsoft Corporation...

powershell# Check if 2023 cert is in db
([System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI db).bytes) -match 'Windows UEFI CA 2023')

# Check factory default db
([System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI dbdefault).bytes) -match 'Windows UEFI CA 2023')

If both return False → you have this exact issue.


⚠️ Note: These PowerShell checks can return false negatives. The cert may be present in binary format the ASCII string match doesn't detect. Run the recovery tool regardless if Secure Boot is blocking your boot.





The Fix

What You Need


A USB drive (any size, even 1GB)
Admin PowerShell access
5 minutes


Step 1 — Confirm the recovery tool is on your machine

powershellTest-Path "C:\Windows\Boot\EFI\securebootrecovery.efi"

Should return True. This is Microsoft's official Secure Boot Recovery tool, built into Windows. No download needed.

Step 2 — Prepare the USB

Format your USB as FAT32, then in admin PowerShell (replace E: with your USB drive letter):

powershellmkdir E:\EFI\BOOT
copy C:\Windows\Boot\EFI\securebootrecovery.efi E:\EFI\BOOT\bootx64.efi

Step 3 — Boot from USB


Enter BIOS (F2 on Lenovo during power-on)
Make sure Secure Boot is disabled
Move USB to first in boot order (F6 to move up)
Save and exit (F10)


Step 4 — Recovery Tool Runs

You'll see:

Microsoft Secure Boot Recovery Version 1.0
Checking Secure Boot Certificate Configuration...
The Secure Boot Certificate database already contains the Microsoft UEFI 2023 certificate.
No changes required. System will reboot in 10 seconds.

The tool either confirms the cert is present or writes it. Either way, you're good.

Step 5 — Re-enable Secure Boot


Remove USB
Enter BIOS again
Move Windows Boot Manager back to first in boot order
Enable Secure Boot
Save and exit


Your system will now boot normally with Secure Boot enabled. ✅


What Doesn't Work (Save Yourself Hours)

MethodWhy It FailsRestore Factory Keys in BIOSdbdefault itself lacks the 2023 cert on affected BIOS versionsbcdboot rebuildNot a boot file issue, it's a cert issueWindows UpdateKB5036210 not delivered to all devices (staged rollout)Registry key AvailableUpdates 0x40Requires KB5036210 as prerequisiteScheduled task Secure-Boot-UpdateSame prerequisite missingSet-SecureBootUEFI manual injectionRequires KEK-signed package; only Microsoft holds the KEK private keyMicrosoft Update Catalog KB5036210No longer listed as standalone update


Affected Hardware (Confirmed)

ModelBIOS VersionStatusLenovo IdeaPad Gaming 3 15IHU6H4CN37WW V2.06✅ Confirmed affected and fixed

If you've confirmed this affects your device with a different model/BIOS, open an issue and I'll add it to the table.


Technical Details

Root cause: InsydeH2O firmware (H4CN37WW V2.06) incorrectly overwrites Secure Boot db on certificate update instead of appending, removing trusted Microsoft Windows Production PCA 2011 cert.

bootmgfw.efi certificate:

Thumbprint: DC91E564D5BC1E3A8E02D6AB508682ABEA8A2443
Issuer: CN=Microsoft Windows Production PCA 2011
Status: Valid (but no longer trusted by db post-overwrite)

Why securebootrecovery.efi works: When Secure Boot is disabled, firmware accepts db writes that would normally require KEK authentication. The recovery tool operates at EFI executable level before Windows loads, allowing it to verify and restore the db state.

Microsoft reference: aka.ms/securebootrecovery


Who Made This

Documented by Mohammed, an 18-year-old cybersecurity self-learner from Karnataka, India, who spent an afternoon diagnosing a firmware-level boot failure. Diagnosed with assistance from Claude — but every command was run, verified, and understood before execution.

If this saved you a trip to a repair shop, drop a ⭐
