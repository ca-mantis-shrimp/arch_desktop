# TODO - Immutable Arch Distribution

## Current Status

✅ **Working**:
- mkosi build system configured and functional
- A/B partition layout with systemd-repart (ESP + root-a + root-b)
- systemd-networkd and systemd-resolved configured
- Base packages installed (base, linux, linux-firmware, systemd, etc.)
- Comprehensive documentation in docs/
- Clean repository structure with proper gitignore
- Build artifacts organized in build/ directory
- Image builds successfully (1.6GB compressed, 18GB raw)

## Critical Issues - RESOLVED AND IDENTIFIED

### ✅ **FIXED: Missing Kernel Command Line**

**Problem**: Boot entry had completely empty kernel command line parameters (`options` field was blank).

**Solution**: Added `KernelCommandLine=root=LABEL=root-a rw console=ttyS0` to mkosi.conf

**Verification**:
- Boot entry now correctly shows: `options root=LABEL=root-a rw console=ttyS0`
- Kernel successfully passes parameters to initrd
- Root partition (LABEL=root-a) is found and mounted to /sysroot

**File changed**: mkosi.conf:25

### ✅ **VERIFIED: Root Filesystem Population**

**Investigation results**:
- Root filesystem (`/dev/loop0p2`, LABEL=root-a) is properly populated
- `/init` symlink exists → `/usr/lib/systemd/systemd`
- `/usr/lib/systemd/systemd` binary present (137KB, executable)
- All system directories present: /etc, /var, /usr, /bin, /sbin, etc.
- All systemd components installed correctly

**Conclusion**: Root filesystem is NOT the issue.

### 🔴 **CURRENT ISSUE: Switch-Root Loop**

**Symptom**: After fixing kernel command line, system still loops infinitely during switch-root:

```
[ OK ] Reached target Switch Root.
      Starting Switch Root...
[systemd restarts and loops back to initrd]
```

**Root Cause Analysis**:

The current A/B partition configuration conflicts with modern systemd/mkosi architecture:

**Current (problematic) approach**:
- ESP: `CopyFiles=/boot:/` and `CopyFiles=/efi:/`
- root-a: `CopyFiles=/` (copies entire system to root partition)
- root-b: empty (for future A/B updates)
- Uses traditional monolithic root partition model

**Recommended systemd approach** (per https://systemd.io/BUILDING_IMAGES/):
- Ship immutable `/usr/` partition (A and B versions)
- Small writable root filesystem created at runtime by systemd-repart
- `/etc`, `/var`, `/home` separate from `/usr`
- A/B updates happen at `/usr` level, not full root
- Use systemd-sysext for additional software layers

## Next Steps (Priority Order)

### Path A: Fix Current Monolithic Root Setup (Quick, Non-Standard)

If you want to get something booting ASAP with the current architecture:

1. **Remove custom mkosi.repart/ entirely**
   - Let mkosi use default single-root partition layout
   - Sacrifice A/B partitions temporarily to verify boot works
   - Test: `mv mkosi.repart mkosi.repart.disabled && mkosi build`

2. **Debug switch-root failure with systemd debug logging**
   ```bash
   # Add to mkosi.conf KernelCommandLine:
   KernelCommandLine=root=LABEL=root-a rw console=ttyS0 systemd.log_level=debug systemd.log_target=console
   ```

3. **Try Unified Kernel Image (UKI)**
   - May have better initrd/root integration
   - Add to mkosi.conf: `UnifiedKernelImages=yes`
   - UKI bundles kernel + initrd + cmdline into single .efi file

### Path B: Adopt Modern systemd Architecture (Recommended, Standard)

Redesign to match systemd best practices for immutable systems:

1. **Study reference implementations**
   - https://0pointer.net/blog/fitting-everything-together.html
   - https://systemd.io/BUILDING_IMAGES/
   - Look for mkosi examples using `/usr` partitions

2. **Redesign partition layout**
   ```
   # New structure:
   - ESP (EFI System Partition)
   - usr-a (immutable, verity-protected /usr partition A)
   - usr-b (immutable, verity-protected /usr partition B)
   - root (writable, created at runtime by systemd-repart)
   - home (optional, created at runtime)
   - var (optional, created at runtime)
   ```

3. **Configure mkosi for /usr-only images**
   - Research mkosi options for building `/usr` trees
   - Set up systemd-repart definitions for runtime partition creation
   - Ship repart definitions IN the image at `/usr/lib/repart.d/`

4. **Implement A/B updates properly**
   - Use GPT partition attributes to mark active/inactive `/usr` partitions
   - systemd automatically boots from partition without GUID:63 attribute
   - Updates write to inactive partition, flip attribute, reboot

5. **Add dm-verity protection**
   - Cryptographically verify `/usr` partition integrity
   - Embed root hash in kernel command line
   - Prevents tampering with system files

### Path C: Hybrid Approach (Pragmatic)

Start simple, migrate to proper architecture later:

1. **Get basic single-root booting first** (Path A, step 1)
2. **Once booting, add second root partition manually**
3. **Implement basic A/B switching with scripts**
4. **Later migrate to `/usr`-based architecture** (Path B)

### Immediate Action Items

**Choose a path** based on your goals:
- **Quick prototype**: Path A or C
- **Production-ready immutable OS**: Path B
- **Learning systemd properly**: Path B

**Current blockers to resolve**:
- ❌ Switch-root infinite loop
- ❓ Unknown: Why switch-root fails with populated root filesystem

**Required investigation**:
- Mount root-a and check `/etc/fstab` (should be empty/absent)
- Verify no conflicting init systems
- Check for /sysroot mount issues in initrd logs

## Investigation Commands

```bash
# Check partition table
fdisk -l build/immutable-arch_0.1.0.raw

# Inspect build output
less build/build.log

# Try alternative boot with more debugging
qemu-system-x86_64 \
  -machine type=q35,accel=kvm \
  -cpu host \
  -m 2G \
  -kernel build/immutable-arch_0.1.0.vmlinuz \
  -initrd build/immutable-arch_0.1.0.initrd \
  -append "root=LABEL=root-a rw init=/usr/lib/systemd/systemd console=ttyS0 systemd.log_level=debug" \
  -drive file=build/immutable-arch_0.1.0.raw,format=raw,if=virtio \
  -nographic

# Check mkosi documentation for partition handling
mkosi --help | grep -i partition
```

## Resources

### Essential Documentation
- [systemd: Building Images](https://systemd.io/BUILDING_IMAGES/) - Official guide for OS image creation
- [Fitting Everything Together](https://0pointer.net/blog/fitting-everything-together.html) - Lennart's comprehensive architecture overview
- [mkosi Re-introduction](https://0pointer.net/blog/a-re-introduction-to-mkosi-a-tool-for-generating-os-images.html) - Modern mkosi usage guide
- [Arch Linux Rescue Image with mkosi](https://swsnr.de/archlinux-rescue-image-with-mkosi/) - Arch-specific mkosi example

### Reference Documentation
- [mkosi GitHub](https://github.com/systemd/mkosi) - Main repository and docs
- [systemd-repart](https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html) - Partition management
- [systemd-boot](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html) - Boot loader
- [Discoverable Partitions Spec](https://systemd.io/DISCOVERABLE_PARTITIONS/) - GPT partition type UUIDs
- [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide) - Base Arch reference

### Key Findings from Investigation
- **A/B updates in systemd**: Use GPT partition attributes, not manual partition management
- **mkosi + A/B**: Issue #379 on GitHub - mkosi delegates A/B to systemd-repart, not build-time config
- **/usr vs root**: Modern immutable systems ship `/usr` partitions, not full root
- **CopyFiles=/**: Copies files at build time; conflicts when both ESP and root try to copy `/boot`

## Notes

- Remember: NEVER test deployment commands on the host system
- Always use VM for testing
- Keep build/ directory in .gitignore
- Document any fixes in docs/BUILD.md
