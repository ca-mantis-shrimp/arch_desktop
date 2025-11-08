# Deploying to New Hardware

This guide covers deploying the immutable Arch distribution to new computers (desktop, homelab, cloud VMs).

## Target Hardware

This distribution is designed for:
- **Desktop**: Intel CPU, NVIDIA GPU, integrated network
- **Homelab**: AMD APU
- **Cloud VMs**: Major cloud providers

All systems assume UEFI boot and a separate systemd-homed partition.

## Prerequisites

1. **Built image**: `immutable-arch_*.raw` from mkosi build
2. **Installation medium**: USB drive (8GB+ recommended)
3. **Target system**: UEFI-capable system with at least 20GB free space
4. **Backup**: Full backup of any existing data!

## Deployment Methods

### Method 1: Direct Disk Write (Simplest)

This method writes the image directly to the target disk. Use for fresh installations.

#### Step 1: Create Bootable Installation Media

From your build system:

```bash
# Compress the image for transfer
xz -9 -T0 immutable-arch_*.raw

# Write to USB drive (ADJUST /dev/sdX to your USB device!)
# WARNING: This will DESTROY all data on the target device!
sudo dd if=immutable-arch_*.raw.xz | xz -d | sudo dd of=/dev/sdX bs=4M status=progress oflag=sync
```

**CRITICAL**: Replace `/dev/sdX` with your actual USB device (use `lsblk` to identify).

#### Step 2: Boot from USB and Write to Target Disk

1. Boot the target system from a live USB (Arch ISO or similar)
2. Identify the target disk:
   ```bash
   lsblk
   ```

3. Write the image:
   ```bash
   # If you have the image on another USB or network location
   dd if=/path/to/immutable-arch_*.raw of=/dev/nvme0n1 bs=4M status=progress oflag=sync
   ```

   Replace `/dev/nvme0n1` with your target disk.

4. Verify partitions:
   ```bash
   lsblk /dev/nvme0n1
   gdisk -l /dev/nvme0n1
   ```

   You should see:
   - Partition 1: EFI System (1GB)
   - Partition 2: Linux root (root-a, 8GB)
   - Partition 3: Linux root (root-b, 8GB)

5. Reboot and remove installation media

### Method 2: systemd-repart on Running System (Advanced)

This method uses systemd-repart to partition and install. Better for complex setups.

**WARNING**: Only use this if you know what you're doing!

#### Step 1: Prepare Partition Definitions

Copy the repart definitions to the target system:

```bash
sudo mkdir -p /etc/repart.d
sudo cp mkosi.repart/*.conf /etc/repart.d/
```

#### Step 2: Run systemd-repart

```bash
# DRY RUN FIRST!
sudo systemd-repart --dry-run=yes /dev/nvme0n1

# If it looks correct, apply:
sudo systemd-repart /dev/nvme0n1
```

#### Step 3: Install System to Root Partitions

```bash
# Mount root-a partition
sudo mount /dev/nvme0n1p2 /mnt

# Extract system files (you'll need the built rootfs)
sudo tar -xzf rootfs.tar.gz -C /mnt

# Install bootloader
sudo bootctl --esp-path=/mnt/efi --boot-path=/mnt/boot install
```

### Method 3: Cloud VM Deployment

For cloud providers:

#### Step 1: Convert Image Format

```bash
# For AWS (convert to AMI)
qemu-img convert -f raw -O vpc immutable-arch_*.raw immutable-arch.vhd

# For Azure
qemu-img convert -f raw -O vpc immutable-arch_*.raw immutable-arch.vhd

# For GCP
qemu-img convert -f raw -O raw immutable-arch_*.raw disk.raw
tar -czf immutable-arch.tar.gz disk.raw
```

#### Step 2: Upload to Cloud Provider

Follow your cloud provider's custom image upload process:
- **AWS**: Use `aws ec2 import-image`
- **GCP**: Use `gcloud compute images create`
- **Azure**: Use `az image create`

## Post-Deployment Configuration

After booting into your new system:

### 1. Change Root Password

```bash
passwd
```

### 2. Set Correct Timezone

```bash
timedatectl set-timezone America/New_York  # Adjust to your timezone
timedatectl list-timezones  # List available timezones
```

### 3. Set Hostname

```bash
hostnamectl set-hostname your-hostname
```

### 4. Configure Networking

The system uses systemd-networkd. Basic DHCP is already configured.

For static IP:

```bash
# Edit /etc/systemd/network/20-wired.network
sudo nano /etc/systemd/network/20-wired.network
```

Example static configuration:
```ini
[Match]
Name=eno1

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=1.1.1.1
DNS=8.8.8.8
```

Restart networking:
```bash
sudo systemctl restart systemd-networkd
```

### 5. Set Up systemd-homed (Your Home Partition)

Since you have a dedicated NVME home partition:

```bash
# Identify your home partition
lsblk

# If it's /dev/nvme1n1p1 for example:
sudo homectl create your-username \
    --storage=directory \
    --image-path=/dev/nvme1n1p1 \
    --real-name="Your Full Name"
```

Or if you're migrating existing homed:

```bash
# Mount your existing home partition
sudo mount /dev/nvme1n1p1 /home

# Activate existing homed users
homectl activate your-username
```

### 6. Add Your User to Sudoers

```bash
# Add your user to wheel group
usermod -aG wheel your-username

# Ensure wheel group has sudo access
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

### 7. Install AUR Helper (paru)

```bash
# Clone and build paru
cd /tmp
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

### 8. Configure NVIDIA Drivers (Desktop Only)

```bash
# Install NVIDIA drivers
sudo pacman -S nvidia nvidia-utils nvidia-settings

# For systemd-boot, add kernel parameters
# Edit /boot/loader/entries/*.conf
# Add to options line:
# nvidia-drm.modeset=1

# Regenerate initramfs
sudo mkinitcpio -P
```

### 9. Set Up Flatpak

```bash
sudo pacman -S flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

### 10. Set Up Podman

```bash
sudo pacman -S podman podman-compose
```

## A/B Update Workflow

The system has two root partitions for atomic updates:

### Current Active Partition

```bash
findmnt /
```

### Updating to the Alternate Partition

1. **Build new image** on your build system
2. **Copy to inactive partition**:
   ```bash
   # Determine which partition is inactive
   # If booted from root-a (/dev/nvme0n1p2), update to root-b (/dev/nvme0n1p3)

   sudo mount /dev/nvme0n1p3 /mnt
   sudo dd if=immutable-arch_new.raw of=/dev/nvme0n1p3 bs=4M status=progress
   ```

3. **Configure systemd-boot** to boot from the new partition:
   ```bash
   # Edit boot entries or use systemd-boot menu at boot
   ```

4. **Reboot and test**

5. **Rollback if needed**: Simply boot from the old partition using systemd-boot menu

## Hardware-Specific Notes

### Desktop (Intel + NVIDIA)

- Install `nvidia` and `nvidia-utils`
- Configure X11 or Wayland accordingly
- May need `nvidia-drm.modeset=1` kernel parameter

### Homelab (AMD APU)

- AMD drivers are in kernel, no extra setup needed
- Consider installing `amdgpu` firmware if not included

### Cloud VMs

- Install cloud-init: `sudo pacman -S cloud-init`
- Configure for your cloud provider
- May need specific kernel modules

## Expanding Partitions

The default partitions are:
- ESP: 1GB
- root-a: 8GB
- root-b: 8GB

To use remaining disk space, add additional partitions:

```bash
# Use gdisk or systemd-repart to add data partitions
sudo gdisk /dev/nvme0n1

# Or add a new partition definition in /etc/repart.d/
```

## Immutability and systemd-sysext

For additional software layers without modifying root:

```bash
# Create sysext overlay
sudo mkdir -p /var/lib/extensions/my-extension
sudo mkosi --distribution=arch --format=directory --output=/var/lib/extensions/my-extension build

# Activate extensions
sudo systemd-sysext refresh
```

## Troubleshooting Deployment

### System Won't Boot

1. Check UEFI settings (Secure Boot may need to be disabled)
2. Verify boot entries: `efibootmgr -v`
3. Manually boot from EFI shell

### No Network After Boot

```bash
# Check networkd status
systemctl status systemd-networkd

# Check interface names
ip link

# Update network config with correct interface name
```

### Can't Login

- Default credentials: root/root
- Boot with `init=/bin/bash` kernel parameter to reset password

### systemd-homed Not Working

```bash
# Check service status
systemctl status systemd-homed

# Check journal
journalctl -u systemd-homed
```

## Backup and Recovery

### Backup Active Partition

```bash
# While booted from root-a, backup root-b:
sudo dd if=/dev/nvme0n1p3 of=/backup/root-b-backup.img bs=4M status=progress
```

### Full System Backup

```bash
# Backup entire disk
sudo dd if=/dev/nvme0n1 of=/backup/full-disk-backup.img bs=4M status=progress
```

## Next Steps

1. **Configure systemd-sysext** for layered software
2. **Set up automatic updates** with systemd timers
3. **Configure portable services** for containerized apps
4. **Integrate with chezmoi** for home directory management
5. **Set up monitoring** and logging as needed

## Security Hardening

Before exposing to network:
1. Set strong passwords for all users
2. Configure SSH with key-based auth only
3. Set up firewall (nftables/firewalld)
4. Enable and configure fail2ban
5. Consider enabling SecureBoot
6. Regular security updates

## Support

For issues specific to this distribution, check:
- Build logs
- `journalctl -b` for boot logs
- `systemctl status` for service issues
