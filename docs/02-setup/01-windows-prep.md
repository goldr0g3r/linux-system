[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: Setup Intro](README.md) · **2.1 Windows Prep** · [Next: 2.2 BIOS →](02-bios-uefi.md)

---

# 2.1 Pre-install: Windows 11 Prep

Do this **in Windows**, before you touch the BIOS or boot a live USB. These steps ensure Windows releases the disk cleanly and does not demand a BitLocker recovery key after you change firmware settings.

## Disable hibernation and Fast Startup

Open an **Administrator PowerShell** in Windows:

```powershell
# Disable hibernation and Fast Startup so Windows releases the disk cleanly.
powercfg /h off

# Confirm it is off.
powercfg /a

# If BitLocker is on the Windows drive, suspend it for 1 reboot so the TPM does not demand a recovery key after the firmware change.
manage-bde -protectors -disable C: -RebootCount 1
```

Windows settings UI (alternative route to disable Fast Startup):

**Settings → System → Power → Additional power settings → Choose what the power buttons do → Change settings that are currently unavailable → uncheck "Turn on fast startup"**.

## Why this matters

- **Fast Startup** leaves the NTFS filesystem in a half-hibernated state. If Linux ever needs to read the Windows disk (it will not in our dual-SSD setup, but belt-and-braces), it refuses because the FS is dirty.
- **Hibernation (`powercfg /h off`)** is redundant with Fast Startup but worth killing outright — it creates `hiberfil.sys`, which wastes a chunk of the Windows SSD and causes the same "dirty FS" problem as Fast Startup.
- **BitLocker suspension** prevents Windows from demanding your 48-digit recovery key after you change Secure Boot state or boot order in BIOS. The `-RebootCount 1` means BitLocker auto-resumes after one reboot; you do not need to re-enable it manually.

## Verification

In the same PowerShell, confirm all three are done:

```powershell
# Hibernation off: "Hibernation has not been enabled" in the output
powercfg /a

# BitLocker suspended (or off): Protection Status: "Protection Off"
manage-bde -status C:
```

If BitLocker shows `Protection On`, either re-run the `-disable` command or power through — worst case you enter the recovery key once.

## Windows is now ready

Close PowerShell. Leave Windows running; the next step is entering BIOS, which happens on the next reboot.

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: Setup Intro](README.md) · **2.1 Windows Prep** · [Next: 2.2 BIOS →](02-bios-uefi.md)
