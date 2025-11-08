# Quickstart Guide

A quick reference for common tasks with the immutable Arch distribution.

## Build Commands

```bash
# Check configuration
mkosi summary

# Build image
mkosi build

# Clean and rebuild
mkosi clean && mkosi build

# Build with specific version
mkosi --image-version=1.0.0 build

# Test in VM
mkosi qemu
```

## VM Testing Credentials

- **Username**: `root`
- **Password**: `root`

## Project Files

```
mkosi.conf              # Main configuration
mkosi.repart/          # Partition definitions (A/B setup)
mkosi.extra/           # Custom files for the image
mkosi.postinst         # Post-installation script
BUILD.md               # Detailed build instructions
DEPLOY.md              # Deployment guide
```

## Key systemd Technologies Used

| Technology | Purpose | Status |
|------------|---------|--------|
| systemd-boot | EFI bootloader | ✓ Configured |
| systemd-repart | Partition management | ✓ A/B partitions |
| systemd-networkd | Network management | ✓ Enabled |
| systemd-resolved | DNS resolution | ✓ Enabled |
| systemd-sysext | Layered extensions | Ready to use |
| systemd-homed | Home directories | Ready to configure |
| Portable services | Container services | Ready to use |

## Partition Layout

| Partition | Size | Label | Purpose |
|-----------|------|-------|---------|
| 1 | 1GB | ESP | EFI System Partition |
| 2 | 8GB | root-a | Active root partition |
| 3 | 8GB | root-b | Backup root for updates |

## Common Workflow

### 1. Development

```bash
# Edit configuration
nano mkosi.conf

# Rebuild
mkosi build

# Test in VM
mkosi qemu
```

### 2. Adding Packages

Edit `mkosi.conf`:
```ini
Packages=
        # ... existing packages ...
        new-package-name
```

Then rebuild.

### 3. Custom Configuration Files

```bash
# Add files to mkosi.extra/ following root structure
mkosi.extra/etc/your-config-file

# Rebuild to include them
mkosi build
```

### 4. Post-Install Customization

Edit `mkosi.postinst` script and rebuild.

## Testing Checklist

In the VM, verify:

```bash
# System status
systemctl status

# Network connectivity
networkctl status
ping -c 3 archlinux.org

# DNS resolution
resolvectl status

# Partition layout
lsblk

# Services
systemctl status systemd-networkd
systemctl status systemd-resolved
systemctl status sshd

# Package manager
pacman -Sy
```

## Deployment Quick Reference

### Create Installation USB

```bash
# Compress image
xz -9 -T0 immutable-arch_*.raw

# Write to USB (CAREFUL: check device name!)
sudo dd if=immutable-arch_*.raw.xz bs=4M status=progress | xz -d | sudo dd of=/dev/sdX oflag=sync
```

### Write to Target Disk

```bash
# From live environment
dd if=immutable-arch_*.raw of=/dev/nvme0n1 bs=4M status=progress oflag=sync
```

### First Boot Configuration

```bash
# Change root password
passwd

# Set timezone
timedatectl set-timezone America/New_York

# Set hostname
hostnamectl set-hostname my-hostname

# Create user with systemd-homed
homectl create username --real-name="Full Name"

# Add user to wheel group
usermod -aG wheel username
```

## A/B Updates

```bash
# Check current root
findmnt /

# Update inactive partition (example: currently on root-a)
sudo mount /dev/nvme0n1p3 /mnt  # Mount root-b
sudo dd if=new-image.raw of=/dev/nvme0n1p3 bs=4M status=progress

# Reboot and select alternate partition from systemd-boot menu
```

## Troubleshooting

### Build fails
```bash
# Check dependencies
pacman -Q mkosi arch-install-scripts

# Check logs
mkosi build --debug
```

### VM won't boot
```bash
# Verify image
file immutable-arch_*.raw

# Try manual QEMU
qemu-system-x86_64 -drive file=immutable-arch_*.raw,format=raw -m 4G
```

### Network issues in deployed system
```bash
# Check networkd
systemctl status systemd-networkd
journalctl -u systemd-networkd

# List interfaces
ip link
networkctl list

# Check configuration
cat /etc/systemd/network/*.network
```

## Next Steps

After successful deployment:

1. **Install desktop environment** (if needed):
   ```bash
   sudo pacman -S gnome
   sudo systemctl enable gdm
   ```

2. **Set up flatpak**:
   ```bash
   sudo pacman -S flatpak
   flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
   ```

3. **Install AUR helper**:
   ```bash
   cd /tmp
   git clone https://aur.archlinux.org/paru.git
   cd paru
   makepkg -si
   ```

4. **Configure NVIDIA** (desktop):
   ```bash
   sudo pacman -S nvidia nvidia-utils
   ```

5. **Set up systemd-sysext** for additional layers

## Resources

- [Arch Wiki: mkosi](https://wiki.archlinux.org/title/Mkosi)
- [systemd-boot](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html)
- [systemd-repart](https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html)
- [systemd-sysext](https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html)

## File Reference

### mkosi.conf Sections

- `[Distribution]` - Target OS settings
- `[Output]` - Image format and compression
- `[Content]` - Packages and bootloader
- `[Validation]` - Security settings
- `[Host]` - Build optimization

### Important Directories

- `/etc/systemd/network/` - Network configs
- `/var/lib/extensions/` - systemd-sysext overlays
- `/var/lib/portables/` - Portable services
- `/boot/` - Kernel and boot files
- `/efi/` - EFI system partition

## Safety Reminders

- ⚠️ ALWAYS test in VM first
- ⚠️ NEVER run `dd` without triple-checking the target device
- ⚠️ Always have backups before deployment
- ⚠️ Change default root password immediately
- ⚠️ This will repartition the target disk (data loss!)
