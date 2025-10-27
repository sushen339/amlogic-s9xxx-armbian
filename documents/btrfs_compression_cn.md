# Btrfs 文件系统压缩级别配置说明

## 概述

当编译 btrfs 文件系统类型的 Armbian 固件时，系统默认使用 `zstd:6` 作为压缩算法和级别。本文档介绍如何修改 btrfs 文件系统的压缩级别。

## 压缩级别说明

### Zstd 压缩算法

- **算法名称**: zstd (Zstandard)
- **压缩级别范围**: 1-15
- **默认级别**: 6
- **特点**:
  - 低级别 (1-3): 更快的压缩/解压速度，较低的压缩率，占用较少 CPU
  - 中级别 (4-9): 平衡压缩率和速度，适合大多数场景
  - 高级别 (10-15): 更高的压缩率，但占用更多 CPU 和时间

### 性能对比

| 压缩级别 | 压缩率 | CPU 占用 | 速度 | 推荐场景 |
|---------|--------|---------|------|---------|
| 1-3     | 低     | 低      | 快   | 性能优先，存储空间充足 |
| 4-6     | 中     | 中      | 中   | 平衡性能和空间（默认推荐）|
| 7-9     | 高     | 高      | 慢   | 空间优先，CPU 性能好 |
| 10-15   | 很高   | 很高    | 很慢 | 极限压缩，不推荐日常使用 |

## 修改方法

### 方法一：修改 rebuild 脚本配置变量（推荐）

编辑 `rebuild` 脚本，找到第 110-115 行左右的配置区域：

```bash
# Set ROOTFS partition file system type, options: [ ext4 / btrfs ]
rootfs_type="ext4"
# Set btrfs compression level (Only effective when rootfs_type is btrfs)
# Compression algorithm: zstd, Compression level: 1-15 (default is 6)
# Higher values provide better compression but use more CPU
btrfs_compress="zstd:6"
```

**修改步骤**：

1. 将 `rootfs_type` 设置为 `"btrfs"` （如果还没有设置）
2. 修改 `btrfs_compress` 的值，例如：
   - 使用压缩级别 3: `btrfs_compress="zstd:3"`
   - 使用压缩级别 9: `btrfs_compress="zstd:9"`
   - 使用压缩级别 1: `btrfs_compress="zstd:1"`

### 方法二：使用命令行参数

在执行 rebuild 脚本时，使用 `-t` 参数指定文件系统类型：

```bash
sudo ./rebuild -t btrfs
```

然后在执行前，先修改脚本中的 `btrfs_compress` 变量值。

## 配置生效的位置

修改 `btrfs_compress` 变量后，压缩级别会自动应用到以下 4 个位置：

### 1. 挂载分区时的选项（第 154 行）
```bash
mount -t ${m_type} -o discard,compress=${btrfs_compress} ${m_dev} ${m_target}
```

### 2. uEnv.txt 启动配置（第 902 行）
```bash
uenv_rootdev="UUID=${ROOTFS_UUID} rootflags=compress=${btrfs_compress} rw rootwait rootfstype=btrfs"
```

### 3. armbianEnv.txt 启动配置（第 905 行）
```bash
armbianenv_rootflags="compress=${btrfs_compress}"
```

### 4. /etc/fstab 挂载配置（第 1012 行）
```bash
fstab_string="defaults,noatime,compress=${btrfs_compress}"
```

## 使用示例

### 示例 1：使用压缩级别 3（性能优先）

```bash
# 编辑 rebuild 脚本
rootfs_type="btrfs"
btrfs_compress="zstd:3"

# 执行编译
sudo ./rebuild -b s905x3 -k 6.1.y
```

### 示例 2：使用压缩级别 9（空间优先）

```bash
# 编辑 rebuild 脚本
rootfs_type="btrfs"
btrfs_compress="zstd:9"

# 执行编译
sudo ./rebuild -b s905x3 -k 6.1.y
```

### 示例 3：编译多个设备使用相同压缩级别

```bash
# 编辑 rebuild 脚本
rootfs_type="btrfs"
btrfs_compress="zstd:6"

# 编译多个设备
sudo ./rebuild -b s905x3_s905d_s912 -k 5.15.y_6.1.y
```

## GitHub Actions 中使用

在 GitHub Actions 工作流文件中（例如 `.github/workflows/build-armbian-server-image.yml`），修改以下部分：

```yaml
- name: Rebuild Armbian
  uses: ophub/amlogic-s9xxx-armbian@main
  with:
    armbian_path: build/output/images/*.img
    armbian_board: s905x3
    armbian_kernel: 6.1.y
    armbian_fstype: btrfs  # 设置文件系统类型为 btrfs
```

然后在仓库中修改 `rebuild` 脚本的 `btrfs_compress` 变量值。

## 注意事项

1. **只对 btrfs 文件系统生效**: 压缩设置仅在 `rootfs_type="btrfs"` 时才会生效
2. **性能影响**: 压缩级别越高，编译和运行时的 CPU 占用越大
3. **存储空间**: 压缩级别越高，生成的镜像和运行时占用的存储空间越小
4. **兼容性**: 所有 Armbian 设备都支持 btrfs 文件系统和 zstd 压缩
5. **建议级别**: 
   - 日常使用推荐: `zstd:6` (默认)
   - 性能优先: `zstd:3` 
   - 空间优先: `zstd:9`

## 常见问题

### Q1: 如何查看当前系统使用的压缩级别？

登录 Armbian 系统后，执行：

```bash
# 查看挂载选项
mount | grep btrfs

# 查看 /etc/fstab 配置
cat /etc/fstab | grep btrfs
```

### Q2: 可以在运行中的系统上修改压缩级别吗？

可以，但只对新写入的数据生效。执行：

```bash
# 重新挂载根分区，使用新的压缩级别
mount -o remount,compress=zstd:9 /

# 永久修改需要编辑 /etc/fstab
sed -i 's/compress=zstd:[0-9]*/compress=zstd:9/g' /etc/fstab
```

### Q3: 不同压缩级别的镜像大小差异有多大？

根据内容不同，压缩率差异约为：
- zstd:1 vs zstd:6: 约 5-10% 的大小差异
- zstd:6 vs zstd:9: 约 3-8% 的大小差异
- zstd:1 vs zstd:15: 约 10-20% 的大小差异

### Q4: 是否支持其他压缩算法？

Btrfs 支持以下压缩算法：
- `zstd`: 推荐使用（默认）
- `lzo`: 速度快，压缩率低
- `zlib`: 压缩率高，速度慢

修改方法相同，例如使用 lzo：
```bash
btrfs_compress="lzo"
```

## 参考资料

- [Btrfs 官方文档](https://btrfs.wiki.kernel.org/)
- [Zstd 压缩算法](https://github.com/facebook/zstd)
- [Armbian 构建文档](README.cn.md)

## 更新记录

- 2024-10-27: 初始版本，添加 btrfs_compress 配置变量
