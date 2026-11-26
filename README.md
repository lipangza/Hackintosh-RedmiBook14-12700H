# RedmiBook 14 (i7-12700H) · OpenCore EFI

适用于 **RedmiBook 14 / 2022 款（i7-12700H + Iris Xe 96EU）** 的 OpenCore 引导配置，当前验证系统为 **macOS 26.7.3 Tahoe**。

- OpenCore：`DBG-109-2026-10-21`（1.0.9 调试版，872,448 B）
- 引导配置：`EFI/OC/config.plist`（73,094 B）
- 验证状态：系统可正常启动、Wi-Fi 原生可用（`AirportItlwm`，非 HeliPort）、Metal 加速需配合系统层 Graphics 包（见下文）
- 更新日期：2026-11-26

## 硬件

| 项目 | 规格 |
| --- | --- |
| CPU | Intel Core i7-12700H（6P + 8E，20 线程） |
| 核显 | Intel Iris Xe 96EU，`8086:46A6` rev `0x0C`（Alder Lake-H） |
| 内屏 | `TL140ADXP20-0`，2880 × 1800，eDP |
| 内存 | 16 GB |
| 硬盘 | 1 TB NVMe（macOS APFS 容器 274.3 GB + Windows 数据区 + ESP 208 MB） |
| 无线 | Intel CNVi（`AirportItlwm 2.4.0` 驱动，系统原生 Airport 接口） |
| 蓝牙 | Intel 集成（kext 已注入，当前未出控制器） |

## 系统

```
macOS 26.7.3 Tahoe · Build 25G342
内核 Darwin 25.6.0 x86_64
SMBIOS: MacBookPro16,2（伪装）
csr-active-config: 0x803（Kext 签名校验关、文件系统保护关）
SecureBootModel: Disabled
```

`csr-active-config = 0x803` 是必要项：WhateverXe 的图形栈依赖它才能在 `/Library/Extensions` 载入签名外的 kext。

## 引导项

UEFI 驱动（`EFI/OC/Drivers`，5 个全启用）

```
OpenRuntime.efi        OpenCore 运行时
apfs_aligned.efi       APFS 读写（EnableJumpstart 已关闭，全靠它，不能删）
HfsPlus.efi            HFS+ 读取
OpenCanopy.efi         图形化选盘界面
ResetNvramEntry.efi    菜单 Reset NVRAM 条目
```

ACPI（`EFI/OC/ACPI`，11 个全启用）

```
SSDT-PLUG-ALT     CPU 电源管理
SSDT-EC           嵌入式控制器
SSDT-PNLF         背光控制（内屏亮度）
SSDT-MCHC         内存控制器
SSDT-USBX         USB 供电参数
SSDT-USB-Reset    USB 复位
SSDT-GPI0         触控板 GPIO
SSDT-ALS0         光线传感器
SSDT-SBUS         总线
SSDT-RTCAWAC      RTC
SSDT-XOSI         OS 识别（Windows 兼容）
```

## 启动参数

```
-v debug=0x100 keepsyms=1
-allow3d -disablegfxfirmware -igfxdvmt
-vi2c-force-polling ctrsmt=full -ibtcompatbeta
revpatch=sbvmm
-wex-panel-handoff -wex-edp-keep -wex-edp-blank -wex-no-edp-hpd -wex-resume-park
```

| 参数 | 作用 |
| --- | --- |
| `-v debug=0x100 keepsyms=1` | 跑码与符号化，排障用；日常可删 |
| `-allow3d` | 允许 3D 加速（WhateverXe 必需） |
| `-disablegfxfirmware` | 禁用核显 GuC 固件加载（必需） |
| `-igfxdvmt` | 绕过 DVMT 预分配限制，配合 `framebuffer-stolenmem` |
| `-vi2c-force-polling` | 强制 I2C 轮询，触控板更稳 |
| `ctrsmt=full` `-ibtcompatbeta` | 蓝牙 / CPU 兼容参数 |
| `revpatch=sbvmm` | 伪装机型版本，便于系统更新 |
| `-wex-panel-handoff` 等 5 条 | eDP 面板移交与背光链路，内屏不亮时必需 |

`DeviceProperties` 中针对 `PciRoot(0x0)/Pci(0x2,0x0)` 设置了：

```
framebuffer-patch-enable = 0x01000000
framebuffer-stolenmem    = 0x0000C003   (64 MB)
```

## 内核驱动

`EFI/OC/Kexts` 共 23 个，`Kernel > Add` 共 30 条注入（29 启用，1 关闭）。

| Kext | 版本 | 作用 | 状态 |
| --- | --- | --- | --- |
| Lilu | 1.7.3 | 补丁注入基础框架 | 启用 |
| VirtualSMC | 1.3.9 | SMC 模拟（必需） | 启用 |
| SMCBatteryManager | 1.3.9 | 电池读数 | 启用 |
| SMCLightSensor | 1.3.9 | 光线传感器 | 启用 |
| SMCProcessor | 1.3.9 | CPU 温度与频率 | 启用 |
| SMCSuperIO | 1.3.9 | 风扇转速 | 启用 |
| AMFIPass | 1.4.1 | AMFI 放行（min 20.0.0） | 启用 |
| RestrictEvents | 1.1.7 | 屏蔽机型相关检查 | 启用 |
| CpuTopologyRebuild | 2.0.2 | 大小核拓扑重排 | 启用 |
| WhateverXe | 0.8.5 | Intel 核显加速主体（min 20.0.0） | 启用 |
| AirportItlwm | 2.4.0 | Intel 无线，原生 Airport 接口 | 启用 |
| itlwm | 2.3.0 | Intel 无线（备选方案） | **关闭**（与 AirportItlwm 互斥） |
| IntelBluetoothFirmware | 2.5.0 | 蓝牙固件上传 | 启用 |
| IntelBTPatcher | 2.5.0 | 蓝牙补丁 | 启用 |
| BlueToolFixup | 2.7.3 | 蓝牙框架修补（min 21.0.0） | 启用 |
| BrightnessKeys | 1.0.4 | 亮度快捷键 | 启用 |
| NVMeFix | 1.1.4 | NVMe 电源管理（min 18.0.0） | 启用 |
| USBToolBox | 1.2.0 | USB 端口映射引擎 | 启用 |
| UTBDefault | 1.0 | 端口映射表 | 启用 |
| XHCI-unsupported | 0.9.2 | 非原生 XHCI 控制器 | 启用 |
| VoodooPS2Controller | 2.3.8 | PS/2 键盘、鼠标 | 启用 |
| VoodooI2C | 2.9.1 | I2C 触控板（含 Services / GPIO / Input） | 启用 |
| VoodooI2CHID | 1 | HID over I2C 触控板主路径 | 启用 |

`VoodooPS2Controller` 内部的 `VoodooInput` 已关闭（由 `VoodooI2C` 接管触控板输入）。

## 系统层组件（不在 EFI 里）

核显真正出加速还需要三个 macOS 系统扩展，由 **WhateverXe Graphics 0.4.5** 安装包写入 `/Library/Extensions`：

```
AppleIntelTGLCompat                0.8.1    org.b00t0x.AppleIntelTGLCompat
AppleIntelTGLGraphics              16.0.0
AppleIntelTGLGraphicsFramebuffer   16.0.0
```

安装包来源：`b00t0x/WhateverXe-Graphics` release `0.4.5`，SHA256 `f6ec2bdcc4d826d9b5be4858da5f4c1b79cc5872ee93ab0ba9abfc3609e9aa37`（95,355,255 B）。

装完后需要重启，系统会重建 `AuxiliaryKernelExtensions` 内核集合。注意：

- 不要安装 **WhateverGreen**，它与 WhateverXe 互斥；
- 不要安装其他核显补丁；
- Tahoe 下若 `kmutil install --update-all` 报 `Missing Developer Kit`，属于正常现象（缺 KDK），交给开机时的自动重建即可。

## 使用方法

1. 挂载目标磁盘的 ESP 分区（macOS：`diskutil mount <ESP 卷 GUID>`；Windows：`mountvol Y: \\?\Volume{GUID}\`）；
2. 用本仓库 `EFI` 覆盖 ESP 内原有 `EFI`；
3. BIOS 关闭 Secure Boot，启动顺序指向该 ESP 的 OpenCore 项；
4. 首次进入 macOS 后安装 WhateverXe Graphics 包，再次重启。

> **序列号提醒**：`config.plist` 内带有 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM`。公开使用前请自行用 GenSMBIOS 重新生成一套，避免与他人冲突。

## 已知问题

1. **蓝牙无控制器**：三个蓝牙 kext 均已加载，但系统报告里蓝牙列表为空，疑为 USB 端口映射未放行蓝牙控制器，或固件未上传。
2. **内屏 120 Hz**：驱动只暴露单一显示模式，系统偏好里的 120 Hz 无法真正套用；改动刷新率可能导致 WindowServer 重启后黑屏。
3. **跑码**：默认开启 `-v`，需要安静启动时删掉 `-v debug=0x100 keepsyms=1`。

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/          11 个 SSDT
    ├── Drivers/       5 个 UEFI 驱动
    ├── Kexts/         23 个 kext
    ├── Resources/     界面资源
    ├── Tools/
    ├── OpenCore.efi
    └── config.plist
```

## 致谢

本配置基于以下开源项目，遵循各自许可证：

- [Acidanthera](https://github.com/acidanthera) — OpenCore、Lilu、VirtualSMC、NVMeFix、RestrictEvents、BrightnessKeys
- [b00t0x/WhateverXe](https://github.com/b00t0x/WhateverXe) 与 [b00t0x/WhateverXe-Graphics](https://github.com/b00t0x/WhateverXe-Graphics) — Intel 核显加速
- [OpenIntelWireless](https://github.com/OpenIntelWireless) — AirportItlwm / itlwm / IntelBluetoothFirmware
- [VoodooI2C](https://github.com/VoodooI2C) — I2C 触控板驱动
- [USBToolBox](https://github.com/USBToolBox) — USB 端口映射
- [b00t0x/CpuTopologyRebuild](https://github.com/b00t0x/CpuTopologyRebuild) — 大小核拓扑
- AMFIPass — AMFI 放行补丁
