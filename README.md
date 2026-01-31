## ⚠️ DO NOT FOLLOW THIS README (DEPRECATED / BROKE SYSTEM)

This document describes an approach I tried to make the SV08’s USB and a network share mount “conveniently” inside:

```
/home/sovol/printer_data/gcodes/
```

**Do not repeat these steps. They break the SV08 UI and can break printing.**

---

### What broke and why

The SV08 (Klipper + Mainsail + Moonraker + KlipperScreen) relies on fast, reliable access to the directory:

```
/home/sovol/printer_data/gcodes/
```

That directory is effectively the printer’s *virtual SD card* (Klipper’s `virtual_sdcard` path). Both the physical screen and the web UI enumerate this directory to list available print files.

This README added **systemd automount (autofs) mountpoints** *inside* that directory:

* `/home/sovol/printer_data/gcodes/USB` (USB drive)
* `/home/sovol/printer_data/gcodes/Network` (CIFS / SMB share)

These mounts were configured via `/etc/fstab` using `x-systemd.automount`.

#### Failure mode

When Linux lists the `gcodes/` directory, `autofs` attempts to resolve and activate those mountpoints. If the USB device is not present, if the CIFS share is slow or unavailable, or if the mount blocks for any reason, the directory listing itself can stall.

This cascades into multiple failures:

* **Physical print menu freeze or long hang** (screen tries to list print files → blocks)
* **Mainsail showing “Not connecting to Moonraker”** when Moonraker’s file manager calls block
* Stale or incorrect file lists

This failure was directly observed: listing `gcodes/` took minutes and showed `d?????????` entries for `USB` and `Network`, indicating unresolved autofs mountpoints.

---

Klipper + Windows Network & USB Setup (Clean Guide)

This guide documents the working setup we built. **All credentials and host details are placeholders**—replace with your values:

- `<HOSTNAME>` — Windows PC name hosting the share (e.g., `LENOVO-YOGA`)
- `<HOST_IP>` — Windows host LAN IP (e.g., `192.168.1.149`)
- `<PI_HOST>` — Klipper Pi hostname (e.g., `SPI-XI`)
- `<PI_IP>` — Klipper Pi LAN IP (e.g., `192.168.1.167`)
- `<username>` — Dedicated share/login user (same user on host + Samba on Pi if you like)
- `<password>` — Password for `<username>`
- `<ShareName>` — Windows share name (we used `gcode`)
- `<DriveLetter>` — Windows drive letter to map (e.g., `Z` or `T`)
- `<USB_DEVICE>` — USB device on the Pi (we used `/dev/sda` because the stick had no partition)

> **Security note**: Use strong passwords; do not leave SMB guest access enabled.

---

## A. Windows Host (shares the folder)

### A1) Create a local user (once)
Create a local user `<username>` with password `<password>` on the host PC.

### A2) Grant NTFS + Share permissions
- Folder path example: `C:\Users\<YourName>\Documents\gcode`
- **NTFS**: Add `<HOSTNAME>\<username>` with **Modify** (or Full Control).
- **Share**: Share as `<ShareName>` (e.g., `gcode`), grant **Change** to `<HOSTNAME>\<username>`.

### A3) Quick PowerShell (Admin) to enforce permissions
```powershell
# --- Edit these ---
$FolderPath = "C:\Users\<YourName>\Documents\gcode"
$ShareName  = "<ShareName>"   # e.g., gcode
$UserName   = "<username>"
# ------------------

$LocalIdentity = "$env:COMPUTERNAME\$UserName"

# NTFS
icacls $FolderPath /grant "${LocalIdentity}:(OI)(CI)M" /T

# Recreate share
Get-SmbShare -Name $ShareName -ErrorAction SilentlyContinue | ForEach-Object {
  Remove-SmbShare -Name $ShareName -Force
}
New-SmbShare -Name $ShareName -Path $FolderPath -ChangeAccess $LocalIdentity

# Firewall and discovery services
Set-Service LanmanServer -StartupType Automatic; Start-Service LanmanServer
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" | Set-NetFirewallRule -Profile Private -Enabled True | Out-Null
Set-Service FDResPub -StartupType Automatic; Start-Service FDResPub
Set-Service FdPHost  -StartupType Automatic; Start-Service FdPHost
```

---

## B. Windows Clients — Auto-map the host share

Run this in a **normal** PowerShell window (per-user, persistent).

```powershell
# === SET THESE ===
$HostIP    = "<HOST_IP>"
$ShareName = "<ShareName>"      # e.g., gcode
$Drive     = "<DriveLetter>"    # e.g., Z
$HostName  = "<HOSTNAME>"
$User      = "<username>"
$Pass      = "<password>"
# ==================

$Target   = "\\$HostIP\$ShareName"
$HostUser = "$HostName\$User"

# Clear old creds and mappings
foreach ($k in @($HostIP, $HostName, $Target)) { try { cmdkey /delete:$k | Out-Null } catch {} }
try { net use "\\$HostIP\*"   /delete /y | Out-Null } catch {}
try { net use "\\$HostName\*" /delete /y | Out-Null } catch {}

# Store correct creds (by IP and by name)
cmdkey /add:$HostIP   /user:$HostUser /pass:$Pass | Out-Null
cmdkey /add:$HostName /user:$HostUser /pass:$Pass | Out-Null

# Map persistently
if (Get-PSDrive -Name $Drive -ErrorAction SilentlyContinue) { net use "${Drive}:" /delete /y | Out-Null }
net use "${Drive}:" "$Target" /persistent:yes
```

---

## C. Klipper Pi — Auto-mount the Windows share into Mainsail

We expose the Windows share at **`~/printer_data/gcodes/Network`**, side by side with USB.

### C1) CIFS + credentials on the Pi
```bash
sudo apt update
sudo apt install -y cifs-utils

# Store SMB credentials safely
sudo bash -c 'cat >/root/.smbcred <<EOF
username=<HOSTNAME>\<username>
password=<password>
EOF'
sudo chmod 600 /root/.smbcred
```

### C2) fstab entry (on-demand automount that stays mounted)
```bash
mkdir -p /home/<pi_user>/printer_data/gcodes/Network
sudo chown -R <pi_user>:<pi_user> /home/<pi_user>/printer_data/gcodes/Network

# Append to /etc/fstab
echo "//<HOST_IP>/<ShareName>  /home/<pi_user>/printer_data/gcodes/Network  cifs  credentials=/root/.smbcred,vers=3.1.1,sec=ntlmssp,uid=<pi_user>,gid=<pi_user>,file_mode=0664,dir_mode=0775,x-systemd.automount,_netdev,nofail  0  0" | sudo tee -a /etc/fstab

# Activate
sudo systemctl daemon-reload
sudo systemctl start home-<pi_user>-printer_data-gcodes-Network.automount

# Verify
findmnt -T /home/<pi_user>/printer_data/gcodes/Network
ls -la  /home/<pi_user>/printer_data/gcodes/Network
```

> If negotiation fails, try `vers=3.0` or `vers=2.1`.

---

## D. Klipper Pi — USB mount that **stays** mounted during prints

Your USB stick presented its filesystem directly on `<USB_DEVICE>` (no partition), so we mounted the **device**. Adjust if yours is `/dev/sda1` etc.

```bash
# Mountpoint
mkdir -p /home/<pi_user>/printer_data/gcodes/USB
sudo chown -R <pi_user>:<pi_user> /home/<pi_user>/printer_data/gcodes/USB

# /etc/fstab line (automount on access; no idle timeout)
echo "<USB_DEVICE>  /home/<pi_user>/printer_data/gcodes/USB  vfat  uid=<pi_user>,gid=<pi_user>,umask=002,flush,shortname=mixed,utf8=1,noauto,x-systemd.automount,_netdev,nofail  0  0" | sudo tee -a /etc/fstab

# Activate
sudo systemctl daemon-reload
sudo systemctl start home-<pi_user>-printer_data-gcodes-USB.automount

# Verify
findmnt -T /home/<pi_user>/printer_data/gcodes/USB
```

**Safe removal:**
```bash
sync && sudo umount /home/<pi_user>/printer_data/gcodes/USB
```

**If the stick shows I/O errors (e.g., on `System Volume Information`), repair:**
```bash
sudo umount /home/<pi_user>/printer_data/gcodes/USB
sudo fsck.vfat -a <USB_DEVICE>   # or run CHKDSK on Windows
```

---

## E. Pi → Share the USB over the network via Samba (`to_print`)

### E1) Install + set a Samba password for `<username>`
```bash
sudo apt update
sudo apt install -y samba smbclient
sudo smbpasswd -a <username>
```

### E2) Define the share **directly in** `/etc/samba/smb.conf`
Append this block at the **end** of the file:
```ini
[to_print]
   path = /home/<pi_user>/printer_data/gcodes/USB
   browseable = yes
   writable  = yes
   guest ok  = no
   valid users = <username>
   force user  = <pi_user>
   force group = <pi_user>
   create mask = 0664
   directory mask = 0775
```

> We place the share in `smb.conf` to avoid include-order pitfalls.

Restart + verify:
```bash
sudo systemctl restart smbd nmbd
testparm -s
smbclient -L localhost -U <username>
```

You should see `to_print` in the share list.

### E3) Map `to_print` on Windows clients
```powershell
net use <DriveLetter>: \\<PI_IP>\to_print /user:<username> <password> /persistent:yes
```

---

## F. Reliability tips

- **Printing from Network share**: It works, but safest is to copy to local storage before long prints (network hiccups can abort prints).
- **Host sleep**: Ensure the Windows host does **not** sleep during prints.
- **Stick format**: Use FAT32 for broad compatibility. Always `sync && umount` before removal on the Pi.
- **DHCP reservations**: Give `<HOST_IP>` and `<PI_IP>` fixed leases in your router.

---

## G. Quick Troubleshooting

- **Windows “System error 67”**: The share name doesn’t exist or Samba didn’t load it. Check `testparm -s`, restart `smbd`, verify the share block.
- **Permission denied**: Use `valid users`, `force user`, and correct NTFS/share perms. Ensure you log in as `<HOSTNAME>\<username>` to Windows host shares.
- **Mount looks empty on Pi**: Confirm `findmnt` shows `cifs`/`vfat`. For SMB, try `vers=3.1.1,sec=ntlmssp`. For USB, run `fsck.vfat`.
- **Mapped drive not visible in Explorer**: Map from a **non-Admin** PowerShell, or enable `EnableLinkedConnections` in the registry if mapped as Admin.

---

**End of guide.**
