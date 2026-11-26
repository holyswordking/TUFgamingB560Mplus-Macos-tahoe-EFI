# TUFgamingB560Mplus-Macos-tahoe-EFI
一、硬件与系统环境

| 项目        | 参数                                                                         |
| ----------- | ---------------------------------------------------------------------------- |
| 主板        | 华硕 TUF GAMING B560M-PLUS                                                   |
| CPU         | Intel Core i3-10100（Comet Lake，3.60 GHz）                                  |
| 独立显卡    | 蓝宝石 Sapphire Nitro+ Radeon RX 580 8GB（Polaris / Baffin，设备 ID 0x67DF） |
| 核显        | Intel UHD Graphics 630（Comet Lake GT2，设备 ID 0x9BC8）                     |
| 内存        | 32 GB                                                                        |
| 显示器      | 5120×2880（5K / UHD Plus，60 Hz）                                            |
| 操作系统    | macOS Tahoe 26.7.3（构建号 25G342）                                          |
| OpenCore    | 0.9.6                                                                        |
| SMBIOS 机型 | iMac20,1                                                                     |
| 驱动版本    | Lilu 1.7.3 / WhateverGreen 1.7.2                                             |

**设计方案**：RX 580 作为主显示卡负责屏幕输出，UHD 630 以 headless（无接口）帧缓冲模式运行，专门承担 H.264 / HEVC 硬件编解码。这是黑苹果上「免驱 AMD 独显 + Intel 核显做计算」的经典组合。
二、最终生效配置速查

### （一）核显注入（DeviceProperties → PciRoot(0x0)/Pci(0x2,0x0)）

| 属性                | base64 值 | 原始字节    | 小端数值   | 说明                              |
| ------------------- | --------- | ----------- | ---------- | --------------------------------- |
| AAPL,ig-platform-id | AwDImw==  | 03 00 C8 9B | 0x9BC80003 | Comet Lake UHD630 headless 帧缓冲 |
| device-id           | yJsAAA==  | C8 9B 00 00 | 0x9BC8     | 与帧缓冲匹配的核显设备 ID         |
| enable-metal        | AQAAAA==  | 01 00 00 00 | 1          | Ventura 及更新系统启用 Metal 3    |
| rps-control         | AQAAAA==  | 01 00 00 00 | 1          | 提升核显性能                      |
| force-online        | AQAAAA==  | 01 00 00 00 | 1          | 优先使用在线渲染器                |

### （二）机型与 board-id（PlatformInfo）

| 字段                        | 最终值               |
| --------------------------- | -------------------- |
| Generic → SystemProductName | iMac20,1             |
| PlatformNVRAM → BID         | Mac-AA95B1DDAB278B95 |
| SMBIOS → BoardProduct       | Mac-AA95B1DDAB278B95 |
| SMBIOS → BoardManufacturer  | Apple Inc.           |

### （三）启动参数（NVRAM → 7C436110-AB2A-4BBB-A880-FE41995C9F82 → boot-args）

```
-v keepsyms=1 debug=0x144 alcid=97 -no_compat_check revpatch=sbvmmm shikigva=1
```

说明：`shikigva=1` 是本轮排障过程中加入的参数，但经核实 WhateverGreen 源码在 macOS 11（Big Sur）及以上只加载 unfair 模块、不加载 shiki 模块，因此该参数在 Tahoe 26 上实际被忽略，属于无效但无害的遗留项，可保留也可删除。`revpatch=sbvmmm` 在 NVRAM 中存在重复设置，同样不影响功能。

### （四）Kext 加载列表（Kernel → Add）

| Kext                        | 状态 | 备注                            |
| --------------------------- | ---- | ------------------------------- |
| Lilu.kext                   | 启用 | 基础注入                        |
| VirtualSMC.kext             | 启用 | SMC 模拟                        |
| SMCProcessor.kext           | 启用 | CPU 传感器                      |
| SMCSuperIO.kext             | 启用 | 超级 I/O 传感器                 |
| WhateverGreen.kext          | 启用 | 显卡/硬解核心                   |
| VPXFixup.kext               | 禁用 | 未启用                          |
| AppleALC.kext               | 启用 | 音频                            |
| NVMeFix.kext                | 启用 | NVMe 优化                       |
| RestrictEvents.kext         | 启用 | 机型限制解除                    |
| itlwm.kext                  | 启用 | Intel Wi-Fi，最低内核 25.0.0    |
| USBToolBox.kext             | 启用 | USB 映射                        |
| UTBMap.kext                 | 启用 | USB 映射数据                    |
| LucyRTL8125Ethernet.kext    | 启用 | 2.5G 有线网卡                   |
| IntelBluetoothFirmware.kext | 启用 | Intel 蓝牙固件，最低内核 25.0.0 |
| IntelBTPatcher.kext         | 启用 | 蓝牙补丁，最低内核 25.0.0       |
| BlueToolFixup.kext          | 启用 | 蓝牙修复，最低内核 25.0.0       |
| SimpleHDA.kext              | 启用 | 音频，最低内核 25.0.0           |

### （五）关键 Quirks（Kernel → Quirks）

| Quirk                   | 值    |
| ----------------------- | ----- |
| AppleCpuPmCfgLock       | True  |
| AppleXcpmCfgLock        | True  |
| DisableIoMapper         | True  |
| PowerTimeoutKernelPanic | True  |
| ProvideCurrentCpuInfo   | False |
| XhciPortLimit           | False |

独显（PciRoot(0x0)/Pci(0x1,0x0)/Pci(0x0,0x0)）另注入了 `PP_PhmSoftPowerPlayTable` 等功耗表与插槽名称属性，用于 RX 580 的功耗管理，本次问题与这部分无关，保持原样
三、关键改动详解
1、补回 headless 三个关键属性
核显注入把`enable-metal`、`rps-control`、`force-online` 三项，否则VDA 失败。tonymacx86 上「UHD630 headless + RX580 主显示」的经典指南明确要求这三项：`enable-metal=1` 在 Ventura 及更新系统启用 Metal 3，`rps-control=1` 提升核显性能，`force-online=1` 优先使用在线渲染器。三项均以 `AQAAAA==`（即数值 1）注入，最终已注入
2、注入正确的 board-id
排查发现，虽然机型设为 iMac20,1，但 `PlatformNVRAM > BID` 和 `SMBIOS > BoardProduct` 两个字段都是空的。这导致运行时 `ioreg` 里 board-id 泄漏成了主板真实型号 `TUF GAMING B560M-PLUS`：

```
"board-id" = <"TUF GAMING B560M-PLUS">
```

AppleGVA（苹果的图形加速框架）是靠 board-id 去 `/System/Library/PrivateFrameworks/AppleGVA.framework/Versions/A/Resources/Info.plist` 的 `System Support` 表里查「这台机器该由谁解码」的。查不到这块主板名，就走默认策略——把 H.264 派给主显卡 RX 580。而 RX 580 是 Polaris 架构，在 macOS Tahoe 26 下的 GVA 硬解已经失效，于是 H.264 不可用、VDA 判定失败。HEVC 则因为 Polaris 原生不支持，系统自动回退到 UHD 630，反而正常可用——这个「HEVC 好、H264 坏」的不对称现象，正是解码器路由错误的指纹。

修复方式是把两个字段都填上 `Mac-AA95B1DDAB278B95`（对应 iMac19,1，真机就是 UHD630 + Polaris 独显，与本机硬件组合高度一致），并把 `BoardManufacturer` 填为 `Apple Inc.`。该 board-id 在 AppleGVA 表中带有 `forceOfflineRenderer=true` 策略，等于强制由核显（offline renderer）接管解码，从而把 H.264 从坏掉的 RX 580 手里拿回来交给 UHD 630。

修复后本机实测确认：

```
"board-id" = <"Mac-AA95B1DDAB278B95">
```
