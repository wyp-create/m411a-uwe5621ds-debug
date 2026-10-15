# M411A (S905L3A) UWE5621DS WiFi on Armbian 6.18

## 硬件
- Box: M411A, Amlogic S905L3A
- WiFi chip: Unisoc UWE5621DS (SDIO, chipid 0x56630001)
- Kernel: Armbian 6.18.53-ophub (mainline)
- DTB: meson-g12a-s905l3a-e900v22c.dtb

## 问题描述
SDIO 卡可识别 (`/sys/bus/sdio/devices/mmc0:8800:1`)，
但 vendor/device ID 读回 0x0000。加载 uwe5621_bsp_sdio.ko 后
`marlin_probe ok`，但 `sdio pt read fail ret:-84` (CRC error)，
最终 `didn't get CP2 version`，wlan0 不出现。

## 已尝试的修复（全部失败）
| 修复项 | 结果 |
|--------|------|
| CMD53 bounce buffer 模式 | CRC 依旧 |
| SDIO 中断重检 (IRQ_SDIO) | 已应用 |
| cap_sdio_irq 条件使能 | 源码自带 |
| MMC_CAP2_SDIO_IRQ_NOTHREAD | 源码自带 |
| 频率降至 12.5MHz | CRC 依旧 |
| 相位组合 2 0 0 / 2 3 0 / 3 0 0 | CRC 依旧 |
| 软复位 meson_mmc_soft_reset | CRC 依旧 |
| 读操作 3 次重试 | 每次读都失败 |
| 电压 3.3V (删 vqmmc-supply) | CRC 依旧 |
| CoreELEC 固件替换 | 无效 |

## 关键修改
1. DTB 加 `wifi@1 { compatible="sprd,unisoc-wifi"; reg=<1>; }`
2. DTB 删 `sd-uhs-sdr50` 加 `sd-uhs-sdr25` `max-frequency=<12500000>`
3. DTB 删 `vqmmc-supply` (否则强制 1.8V)
4. 驱动 `sdiohal_init` 加 `sdiohal_scan_card()` 调用
5. 驱动 Makefile 开 `-DOTT_UWE` 让 sprdwl_ng 自动 probe
6. 内核 `meson-gx-mmc.c` 加 CMD53 bounce buffer + 软复位

## 结论
CRC 错误是持续性、物理层的。所有已知软件修复都试过，指向
Amlogic S905L3A SDIO 控制器与 UWE5621DS 的硬件兼容性问题。

## 最终方案
外接 USB WiFi 网卡 (RTL8811CU / MT7601U)。

## 参考
- ophub/amlogic-s9xxx-armbian Issue #3332 (同类案例)
- ktpeng2026/fn_linux_arm64_uwe5621ds-aml-6.18
- Linux 主线 meson-gx-mmc.c
