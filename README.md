# Skyward for 小米 MIX 2（chiron）

把 [kirklin/skyward](https://github.com/kirklin/skyward) 移植到小米 MIX 2 的仓库：
A-only 分区适配、boot image v0–v2 与 gzip / LZ4 ramdisk 支持，以及在原厂
MIUI recovery 上构建好的可刷镜像。

Skyward 是一个用 Rust 从零写的 recovery：解密 data、刷机、在线更新、备份、
文件管理、终端、adb / MTP，跑的是手机自己的内核驱动与原厂服务。

## 设备

| 项目 | 值 |
| --- | --- |
| 代号 | `chiron` |
| 机型 | 小米 MIX 2 |
| 平台 | 骁龙 835 |
| 分区方案 | A-only，单 boot / recovery，无 super、无 A/B 槽位 |
| 屏幕 | 1080 × 2160，420 dpi |
| recovery 分区 | 64 MiB |
| 原厂基线 | MIUI V12.0.1.0.PDECNXM（Android 9） |

## 直接刷机

预编译镜像在 [Releases](../../releases) 里，是整分区镜像（64 MiB，正好填满
recovery 分区），解压后在 fastboot 下写入：

```sh
sha256sum skyward-chiron-V12.0.1.0.PDECNXM.img
fastboot flash recovery skyward-chiron-V12.0.1.0.PDECNXM.img
fastboot reboot recovery
```

镜像是在原厂 recovery 上盖出来的：原厂的 init、adbd、SELinux 策略、驱动库和
触摸固件都保留，只把 `sbin/recovery` 换成 Skyward，所以显示、触摸、解密走的
是这台手机自己的驱动。

> 需要先解 BL 锁。刷入会覆盖原厂 recovery，MIUI 系统更新可能把它换回去。

## 从源码构建

依赖：Rust stable（版本见 `rust-toolchain.toml`）、Android NDK r29、cargo-ndk、lz4。

```sh
rustup target add aarch64-linux-android
cargo install cargo-ndk
export NDK_HOME=$HOME/Android/Sdk/ndk/29.0.14206865   # 换成你的 NDK 路径
./scripts/build.sh
```

脚本依次做四件事：

1. 用 `--platform 28` 编译设备端 `skyward`（Android 9 只有这些接口）；
2. 加 `crt-static` 静态编译 `skyward-system-init`；
3. 编译主机端打包器 `skyward-pack`；
4. 从 MIUI 卡刷包里只读 recovery 分区那一段取出原厂镜像，再盖成 Skyward。

产物在 `out/skyward-chiron.img`，脚本最后打印大小和 SHA-256。

不想让脚本联网取包，就把 fastboot 包里的 `images/recovery.img` 放到
`stock/recovery.img`，或者用 `STOCK=/path/to/recovery.img ./scripts/build.sh`。

## 这个移植改了什么

相对上游，为跑通 A-only 的老机器改了这些：

- **boot image**：读写 header v0 / v1 / v2，按 header 的 page size 切分，
  保留 second bootloader、DTB、DTBO，写回时重算 SHA-1 / SHA-256 digest。
- **ramdisk**：自动识别 gzip、LZ4 legacy framing 和未压缩三种格式，按原格式压回。
- **A-only 运行时**：新增 `Slot::None`（空后缀），`/dev/block/by-name` 回退到
  `/dev/block/bootdevice/by-name`，fstab 回退读 `/etc/recovery.fstab`，
  安装、备份、脚本刷机都按单分区处理，不再假设 super 和槽位。
- **打包器**：自动识别 `system/bin/recovery` 或 `sbin/recovery`、`prop.default`
  或 `default.prop`；原厂镜像没有 AVB footer 时用 `--partition-size` 指定分区大小。

## 已知限制

- 没有 payload OTA、fastbootd、A/B 切槽和双系统（这台机器本来就没有）。
- 尚未在真机上完整验证；SELinux 沿用原厂 cmdline 的 permissive。
- recovery 分区只有 64 MiB，镜像正好占满，没有余量。

## 许可

[GPL-3.0-or-later](LICENSE)。上游项目：[kirklin/skyward](https://github.com/kirklin/skyward)。
