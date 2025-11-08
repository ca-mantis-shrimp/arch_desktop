# Building the Immutable Arch Distribution

This guide covers building and testing the immutable Arch Linux distribution using mkosi.

## Prerequisites

Ensure you have the following packages installed on your build system:

```bash
sudo pacman -S mkosi arch-install-scripts qemu-full
```

**Important**: This build process should be run on a reflink-capable filesystem (Btrfs or XFS) for optimal performance.

## Project Structure

```
.
├── mkosi.conf              # Main mkosi configuration
├── mkosi.repart/           # Partition definitions for systemd-repart
│   ├── 00-esp.conf        # EFI System Partition (1GB)
│   ├── 10-root-a.conf     # Root partition A (8GB) - active
│   └── 11-root-b.conf     # Root partition B (8GB) - backup for updates
├── mkosi.extra/            # Custom files to include in the image
│   └── etc/systemd/network/  # systemd-networkd configurations
├── mkosi.postinst          # Post-installation script (runs in chroot)
└── BUILD.md               # This file
```

## Configuration Overview

### mkosi.conf

The main configuration creates:
- **Distribution**: Arch Linux (x86-64)
- **Format**: Bootable disk image
- **Bootloader**: systemd-boot
- **Partitions**: A/B update scheme via systemd-repart
- **Compression**: Level 9 for smaller images

### Key Features

1. **A/B Updates**: Two root partitions for atomic updates with rollback capability
2. **systemd-centric**: Uses systemd-networkd, systemd-resolved, systemd-boot, systemd-repart
3. **Immutable design**: Designed for stability with layer-based customization via systemd-sysext
4. **Hardware support**: Intel/AMD CPU microcode, NVIDIA GPU ready (add drivers later)

### Included Packages

- **Base**: base, linux, linux-firmware
- **Microcode**: intel-ucode, amd-ucode
- **Networking**: systemd-networkd, systemd-resolved, iwd
- **Development**: base-devel, git (for AUR/paru later)
- **Tools**: vim, nano, sudo, openssh
- **Filesystems**: btrfs-progs, dosfstools, e2fsprogs, xfsprogs

## Building the Image

### 1. Check Configuration

First, verify your configuration without building:

```bash
mkosi summary
```

This shows what will be built without actually building it.

### 2. Build the Image

Build the bootable disk image:

```bash
mkosi build
```

**Build time**: First build may take 15-30 minutes depending on network speed and CPU.

The output will be:
- `immutable-arch_<version>.raw` - The bootable disk image
- `immutable-arch_<version>.raw.xz` (if compressed) - Compressed version

### 3. Incremental Builds

Thanks to `Incremental=yes` in the config, subsequent builds are much faster:

```bash
mkosi build
```

### 4. Clean Build

To start fresh:

```bash
mkosi clean
mkosi build
```

## Testing in a VM

**CRITICAL**: ALWAYS test in a VM before deploying to real hardware!

### Option 1: Using mkosi's Built-in VM (Recommended)

mkosi includes a built-in QEMU wrapper:

```bash
mkosi qemu
```

This will:
- Boot the image in QEMU
- Set up networking automatically
- Provide a console

**Boot credentials**:
- Username: `root`
- Password: `root` (CHANGE THIS before production use!)

### Option 2: Manual QEMU

For more control:

```bash
qemu-system-x86_64 \
    -machine type=q35,accel=kvm \
    -cpu host \
    -m 4G \
    -smp 4 \
    -drive file=immutable-arch_*.raw,format=raw,if=virtio \
    -bios /usr/share/ovmf/x64/OVMF.fd \
    -net nic,model=virtio \
    -net user,hostfwd=tcp::2222-:22 \
    -nographic
```

Or with graphical output:

```bash
qemu-system-x86_64 \
    -machine type=q35,accel=kvm \
    -cpu host \
    -m 4G \
    -smp 4 \
    -drive file=immutable-arch_*.raw,format=raw,if=virtio \
    -bios /usr/share/ovmf/x64/OVMF.fd \
    -net nic,model=virtio \
    -net user \
    -vga virtio
```

### Testing Checklist in VM

Once booted in the VM, verify:

1. **Boot process**:
   ```bash
   systemctl status
   journalctl -b
   ```

2. **Networking** (systemd-networkd):
   ```bash
   networkctl status
   ping -c 3 archlinux.org
   ```

3. **DNS resolution** (systemd-resolved):
   ```bash
   resolvectl status
   ```

4. **Partitions**:
   ```bash
   lsblk
   findmnt
   ```

5. **Services**:
   ```bash
   systemctl status systemd-networkd
   systemctl status systemd-resolved
   systemctl status sshd
   ```

6. **Package management**:
   ```bash
   pacman -Sy
   ```

## Build Customization

### Changing the Version

```bash
mkosi --image-version=1.0.0 build
```

### Adding More Packages

Edit `mkosi.conf` and add to the `Packages=` section:

```ini
Packages=
        # ... existing packages ...
        your-package-here
```

### Custom Files

Place files in `mkosi.extra/` following the root filesystem structure:

```bash
mkosi.extra/etc/your-config-file
mkosi.extra/usr/local/bin/your-script
```

### Post-Installation Customization

Edit `mkosi.postinst` to add more configuration steps.

## Troubleshooting

### Build Fails with "Command not found"

Ensure `arch-install-scripts` is installed:
```bash
sudo pacman -S arch-install-scripts
```

### Slow Builds

Make sure you're on a reflink filesystem (Btrfs/XFS):
```bash
df -T .
```

### QEMU Won't Boot

Check that your image built successfully:
```bash
file immutable-arch_*.raw
```

Should show: "DOS/MBR boot sector"

### Out of Space

The image requires ~20GB during build. Check available space:
```bash
df -h .
```

## Next Steps

After successful VM testing:
1. Review [DEPLOY.md](./DEPLOY.md) for installation instructions
2. Consider setting up systemd-sysext layers for additional software
3. Configure systemd-homed for your home directory
4. Set up automatic updates with A/B partition switching

## Security Notes

**BEFORE PRODUCTION**:
1. Change the root password (currently: `root`)
2. Consider enabling SecureBoot in mkosi.conf
3. Set up proper user accounts
4. Configure firewall rules
5. Review SSH configuration in `/etc/ssh/sshd_config`
