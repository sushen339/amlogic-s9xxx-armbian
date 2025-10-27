# Btrfs Filesystem Compression Level Configuration Guide

## Overview

When building Armbian firmware with btrfs filesystem, the system uses `zstd:6` as the default compression algorithm and level. This document explains how to modify the btrfs filesystem compression level.

## Compression Level Description

### Zstd Compression Algorithm

- **Algorithm**: zstd (Zstandard)
- **Compression Level Range**: 1-15
- **Default Level**: 6
- **Characteristics**:
  - Low levels (1-3): Faster compression/decompression, lower compression ratio, less CPU usage
  - Mid levels (4-9): Balanced compression ratio and speed, suitable for most scenarios
  - High levels (10-15): Higher compression ratio, but more CPU intensive and slower

### Performance Comparison

| Compression Level | Compression Ratio | CPU Usage | Speed | Recommended Use Case |
|------------------|-------------------|-----------|-------|---------------------|
| 1-3              | Low               | Low       | Fast  | Performance priority, sufficient storage |
| 4-6              | Medium            | Medium    | Medium| Balanced performance and space (default) |
| 7-9              | High              | High      | Slow  | Space priority, good CPU performance |
| 10-15            | Very High         | Very High | Very Slow | Extreme compression, not recommended for daily use |

## How to Modify

### Method 1: Modify the rebuild Script Configuration Variable (Recommended)

Edit the `rebuild` script and find the configuration section around lines 110-115:

```bash
# Set ROOTFS partition file system type, options: [ ext4 / btrfs ]
rootfs_type="ext4"
# Set btrfs compression level (Only effective when rootfs_type is btrfs)
# Compression algorithm: zstd, Compression level: 1-15 (default is 6)
# Higher values provide better compression but use more CPU
btrfs_compress="zstd:6"
```

**Modification Steps**:

1. Set `rootfs_type` to `"btrfs"` (if not already set)
2. Modify the `btrfs_compress` value, for example:
   - Use compression level 3: `btrfs_compress="zstd:3"`
   - Use compression level 9: `btrfs_compress="zstd:9"`
   - Use compression level 1: `btrfs_compress="zstd:1"`

### Method 2: Using Command Line Parameters

When executing the rebuild script, use the `-t` parameter to specify the filesystem type:

```bash
sudo ./rebuild -t btrfs
```

Then modify the `btrfs_compress` variable value in the script before execution.

## Where the Configuration Takes Effect

After modifying the `btrfs_compress` variable, the compression level is automatically applied to the following 4 locations:

### 1. Mount Options When Mounting Partitions (Line 154)
```bash
mount -t ${m_type} -o discard,compress=${btrfs_compress} ${m_dev} ${m_target}
```

### 2. uEnv.txt Boot Configuration (Line 902)
```bash
uenv_rootdev="UUID=${ROOTFS_UUID} rootflags=compress=${btrfs_compress} rw rootwait rootfstype=btrfs"
```

### 3. armbianEnv.txt Boot Configuration (Line 905)
```bash
armbianenv_rootflags="compress=${btrfs_compress}"
```

### 4. /etc/fstab Mount Configuration (Line 1012)
```bash
fstab_string="defaults,noatime,compress=${btrfs_compress}"
```

## Usage Examples

### Example 1: Use Compression Level 3 (Performance Priority)

```bash
# Edit rebuild script
rootfs_type="btrfs"
btrfs_compress="zstd:3"

# Execute build
sudo ./rebuild -b s905x3 -k 6.1.y
```

### Example 2: Use Compression Level 9 (Space Priority)

```bash
# Edit rebuild script
rootfs_type="btrfs"
btrfs_compress="zstd:9"

# Execute build
sudo ./rebuild -b s905x3 -k 6.1.y
```

### Example 3: Build Multiple Devices with Same Compression Level

```bash
# Edit rebuild script
rootfs_type="btrfs"
btrfs_compress="zstd:6"

# Build multiple devices
sudo ./rebuild -b s905x3_s905d_s912 -k 5.15.y_6.1.y
```

## Using in GitHub Actions

In GitHub Actions workflow files (e.g., `.github/workflows/build-armbian-server-image.yml`), modify the following section:

```yaml
- name: Rebuild Armbian
  uses: ophub/amlogic-s9xxx-armbian@main
  with:
    armbian_path: build/output/images/*.img
    armbian_board: s905x3
    armbian_kernel: 6.1.y
    armbian_fstype: btrfs  # Set filesystem type to btrfs
```

Then modify the `btrfs_compress` variable value in the `rebuild` script in the repository.

## Important Notes

1. **Only Effective for btrfs Filesystem**: Compression settings only take effect when `rootfs_type="btrfs"`
2. **Performance Impact**: Higher compression levels result in greater CPU usage during build and runtime
3. **Storage Space**: Higher compression levels result in smaller image sizes and runtime storage usage
4. **Compatibility**: All Armbian devices support btrfs filesystem and zstd compression
5. **Recommended Levels**: 
   - Daily use recommended: `zstd:6` (default)
   - Performance priority: `zstd:3` 
   - Space priority: `zstd:9`

## FAQ

### Q1: How to check the current system's compression level?

After logging into the Armbian system, execute:

```bash
# Check mount options
mount | grep btrfs

# Check /etc/fstab configuration
cat /etc/fstab | grep btrfs
```

### Q2: Can I modify the compression level on a running system?

Yes, but it only affects newly written data. Execute:

```bash
# Remount root partition with new compression level
mount -o remount,compress=zstd:9 /

# For permanent modification, edit /etc/fstab
sed -i 's/compress=zstd:[0-9]*/compress=zstd:9/g' /etc/fstab
```

### Q3: What is the image size difference between different compression levels?

Depending on content, compression ratio differences are approximately:
- zstd:1 vs zstd:6: About 5-10% size difference
- zstd:6 vs zstd:9: About 3-8% size difference
- zstd:1 vs zstd:15: About 10-20% size difference

### Q4: Are other compression algorithms supported?

Btrfs supports the following compression algorithms:
- `zstd`: Recommended (default)
- `lzo`: Fast speed, low compression ratio
- `zlib`: High compression ratio, slow speed

Modification method is the same, for example using lzo:
```bash
btrfs_compress="lzo"
```

## References

- [Btrfs Official Documentation](https://btrfs.wiki.kernel.org/)
- [Zstd Compression Algorithm](https://github.com/facebook/zstd)
- [Armbian Build Documentation](README.md)

## Changelog

- 2024-10-27: Initial version, added btrfs_compress configuration variable
