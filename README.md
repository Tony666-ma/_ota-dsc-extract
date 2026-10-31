# _ota-dsc-extract

在 GitHub Actions 的 **macOS arm64 runner** 上，从 iOS 18.7 (22H20) 全量 OTA 里
展开 `cryptex-system-arm64e` 的 RIDIFF10 编码，取出完整的
`dyld_shared_cache_arm64e` 家族。

## 为什么需要 macOS

`9ab7174531098c763c73a13af25cccac10282535.zip` 里的
`AssetData/payloadv2/image_patches/cryptex-system-arm64e` 是 **RIDIFF10** 容器：

- `pbzx` → 多段 XZ 只能剥掉压缩层，得到 1,408,607,581 字节（**33.0%**）
- 解压流头部 `0x20` 声明输出 **4,273,995,776 字节**（4076 MiB，与
  `AssetData/Info.plist` 的 `CryptexSizeInfo.CryptexSize=4076` 一致）
- 剩下 67% 由 AppleArchive 的 **PrivateStream** 应用，API 只存在于 macOS

`ipsw` 官方文档：

> iOS16.x or macOS13.x or newer this will only work when ran on a **macOS Ventura**
> host as it calls into a **private API** to apply the patch.

## 这个 OTA 是全量包，不是 delta

来自 `https://api.appledb.dev/ios/iOS;22H20.json` 的 `sources`（737 条）：

```
[0] type=ota  size=7130336719  systemImageId=01EA73C8-...
    devices=iPhone14,5
    无 prerequisiteBuild              ← 全量
[27] size=1161551792  prerequisiteBuild=['22E240','22E252']   ← delta
[28] size=780690171   prerequisiteBuild=22F76                ← delta
[29] size=681838821   prerequisiteBuild=['22G84','22G86','22G100']
[30] size=496575391   prerequisiteBuild=22G90
```

本地文件 7,130,336,719 字节，与该条目的 `size` 一致。

## Runner

| 项 | 值 |
|---|---|
| 标签 | `macos-15` |
| 架构 | arm64 |
| 系统 | macOS 15.8.2 (24H232) / Darwin 24.6.0 |
| 要求 | macOS 13+ ✅ |

备选：`macos-26`（macOS 26.6.2）。`macos-14` 已于 2026-11-02 停止支持。

## 运行

```bash
gh workflow run extract-dsc.yml -R Tony666-ma/_ota-dsc-extract
gh run watch -R Tony666-ma/_ota-dsc-extract
```

## 成功判据

`dyld_shared_cache_arm64e` 主文件必须存在，且前 7 字节是 `dyld_v1`。

拿到后在本机：

```bash
py -3.11 -X utf8 tools/dsc_header_dump.py dyld_shared_cache_arm64e
py -3.11 -X utf8 tools/gadget_resolve.py dyld_shared_cache_arm64e \
    --images-header <off> --locate --coverage
```

`gadget_resolve.py` 里那 7 个 gadget 的宿主 image
（HealthRecords / PodcastsFoundation / Charts / UIKitCore /
libcorecrypto / CoreGraphics / Accelerate）的 `__TEXT` 一旦可读，
就可以逐 build 搜索 gadget 指令序列，补齐最后 7 个键。
