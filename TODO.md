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

## Critical Issues

### 🔴 Boot Loop in VM

**Symptom**: System boots into initrd, starts systemd services, but gets stuck in a loop repeatedly switching between initrd stages without completing boot to the real root filesystem.

**Observed behavior**:
- ✅ Kernel loads successfully
- ✅ Initrd unpacks and runs
- ✅ systemd services start (networkd, resolved, udev, etc.)
- ✅ Root partition (LABEL=root-a) is found and mounted to /sysroot
- ✅ Switch root process initiates
- ❌ System loops back to initrd instead of booting into real root

**Potential causes**:
1. **Missing init in root filesystem**
   - The root filesystem may not have /sbin/init or /usr/lib/systemd/systemd
   - Check: Mount the image and verify init exists

2. **Incorrect kernel command line**
   - Current: `root=LABEL=root-a rw console=ttyS0`
   - May need: `init=/usr/lib/systemd/systemd` explicitly
   - May need: `systemd.unit=multi-user.target`

3. **Initrd not properly handing off to root**
   - The switch-root process may be failing
   - Missing files in /sysroot needed for switch-root
   - Check systemd-repart partition population

4. **Root filesystem not properly populated**
   - mkosi.repart/10-root-a.conf has `CopyFiles=/`
   - This may not be working correctly
   - Verify root partition actually has system files

## Next Steps (Priority Order)

### High Priority

1. **Debug the root filesystem contents**
   ```bash
   # Mount the image and inspect
   sudo losetup -fP build/immutable-arch_0.1.0.raw
   sudo mount /dev/loop0p2 /mnt  # Assuming root-a is partition 2
   ls -la /mnt
   ls -la /mnt/usr/lib/systemd/
   # Check for init/systemd binary
   sudo umount /mnt
   sudo losetup -d /dev/loop0
   ```

2. **Fix kernel command line**
   - Try adding explicit init parameter
   - Test: `root=LABEL=root-a rw init=/usr/lib/systemd/systemd console=ttyS0`

3. **Review mkosi.repart configuration**
   - The `CopyFiles=/` directive may need adjustment
   - Consider using mkosi's default partition handling
   - May need to remove custom repart config and let mkosi handle it

4. **Check mkosi build logs for partition population**
   ```bash
   grep -i "copying files" build/build.log
   grep -i "partition" build/build.log
   grep -i "sysroot" build/build.log
   ```

### Medium Priority

5. **Test with simplified configuration**
   - Remove custom mkosi.repart/ temporarily
   - Let mkosi create default partition layout
   - See if that boots successfully

6. **Add debug kernel parameters**
   ```bash
   systemd.log_level=debug systemd.log_target=console
   ```

7. **Consider UKI (Unified Kernel Image)**
   - Current config uses separate kernel/initrd
   - UKI might have better integration
   - Set `UnifiedKernelImages=yes` in mkosi.conf

### Low Priority

8. **Add systemd-homed support**
   - Configure separate home partition
   - Integrate with existing NVME home drive

9. **Set up systemd-sysext layers**
   - Create extension images for additional software
   - Test overlay functionality

10. **Configure NVIDIA drivers**
    - Add to package list
    - Configure kernel modules

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

- [mkosi documentation](https://github.com/systemd/mkosi)
- [systemd-repart](https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html)
- [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [systemd-boot](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html)

## Notes

- Remember: NEVER test deployment commands on the host system
- Always use VM for testing
- Keep build/ directory in .gitignore
- Document any fixes in docs/BUILD.md
