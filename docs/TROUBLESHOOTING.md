# Troubleshooting Notes

## Issue: mkosi boot/shell failing with "No medium found"

### Symptoms
- `mkosi boot` failed with: `Failed to mount image: No medium found`
- `mkosi shell` failed with same error
- `mkosi qemu` worked perfectly fine
- Issue only occurred with `bootable` profile using A/B partition layout

### Root Cause
The problem was creating **both** root-a and root-b partitions at build time in `mkosi.repart/`:

1. **Partition selection logic**: systemd-dissect (used internally by systemd-nspawn) selects between multiple partitions of the same type using `strverscmp()` on partition labels
2. **String comparison**: `"root-b" > "root-a"` in version comparison, so root-b was selected
3. **Empty partition**: root-b was created and formatted but had no content (no `CopyFiles=/` directive)
4. **Mount failure**: systemd-nspawn tried to boot from empty root-b → "No medium found"

### Why qemu worked but boot/shell didn't
- **mkosi qemu**: Boots via QEMU using the kernel on the ESP with `root=PARTUUID` kernel command line - directly specifies which partition to use
- **mkosi boot/shell**: Use systemd-nspawn which relies on systemd-dissect for automatic partition discovery - picks "highest" version via label comparison

### Solution
Removed `mkosi.repart/20-root-b.conf` to follow the canonical systemd A/B update pattern:

- **Build time** (mkosi.repart/): Create only root-a partition with content
- **First boot** (future work): systemd-repart creates root-b partition labeled `_empty`
- **Updates**: systemd-sysupdate manages switching between A and B partitions

### Key Resources That Helped

1. **Discoverable Partitions Specification** (https://uapi-group.org/specifications/specs/discoverable_partitions_specification)
   - Documents how multiple partitions of the same type are selected
   - Explains `strverscmp()` comparison on partition labels for A/B versioning

2. **"Fitting Everything Together" blog** (https://0pointer.net/blog/fitting-everything-together.html)
   - Describes complete A/B update workflow
   - Explains `_empty` label convention for unused partitions
   - Shows version labeling pattern (e.g., `fooOS_0.7`)

3. **mkosi issue #379** (https://github.com/systemd/mkosi/issues/379)
   - Confirmed A/B partitions should be created at boot time, not build time
   - Explained preference for smaller images that adapt to hardware

4. **systemd.io/BUILDING_IMAGES/** (https://systemd.io/BUILDING_IMAGES/)
   - Official guidance: "ship the original A file system in the deployed image, but create the B partition on first boot"

### What Would Have Been Faster

1. **Run `systemd-dissect` immediately**: `sudo systemd-dissect mkosi.output/image.raw` would have instantly shown which partition was being selected and why

2. **Understand the two-phase configuration**:
   - `mkosi.repart/` = build-time partition creation (minimalist)
   - `/usr/lib/repart.d/` in the image = runtime partition management (dynamic growth/A-B creation)

3. **Check man pages systematically**:
   - `man systemd-dissect` - partition discovery rules
   - `man systemd-nspawn` - how --image works
   - `man systemd-repart` - Label= directive and partition configuration

4. **Recognize the difference in boot paths**:
   - systemd-nspawn = container with partition auto-discovery
   - QEMU = VM with explicit kernel command line

### Commands for Future Debugging

```bash
# See what systemd-dissect discovers and which partition it selects
sudo systemd-dissect mkosi.output/image.raw

# Check partition table details
sudo fdisk -l mkosi.output/image.raw
sudo sfdisk -d mkosi.output/image.raw

# Test systemd-nspawn directly
sudo systemd-nspawn --image=mkosi.output/image.raw --boot

# Check what's actually in the image
sudo systemd-dissect --with mkosi.output/image.raw ls /
```

### Lessons Learned

- **Do research before guessing**: Went down privilege/authentication rabbit hole before understanding the actual partition selection problem
- **Trust the tools**: systemd-dissect output clearly showed root-b was selected - should have run that first
- **Read the specs**: The Discoverable Partitions Specification had the answer about `strverscmp()` behavior
- **Follow canonical patterns**: systemd developers designed A/B updates to work a specific way - don't fight it
