---
title: Linux 硬件信息查询命令备忘录
published: 2026-05-31
description: Linux 硬件查看/调试命令汇总
tags: [dev, perf, OS]
category: 调试工具
draft: false
---

## 系统
```sh
uname -a
# Linux dragonos-gtx1065 5.15.0-177-generic #187-Ubuntu SMP Sat Apr 11 22:54:33 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux

hostnamectl
#  Static hostname: mycomputer-gtx1065
#        Icon name: computer-desktop
#          Chassis: desktop
#       Machine ID: 8ea88b201cd0486dad6c2be6f1e91bad
#          Boot ID: e3194afb773e4969b46d279436128c21
# Operating System: Ubuntu 22.04.5 LTS
#           Kernel: Linux 5.15.0-177-generic
#     Architecture: x86-64
#  Hardware Vendor: Micro-Star International Co., Ltd.
#   Hardware Model: MS-7C96
```

## baseboard
```sh
sudo dmidecode -t baseboard
# Getting SMBIOS data from sysfs.
# Handle 0x0002, DMI type 2, 15 bytes
# Base Board Information
#         Manufacturer: Micro-Star International Co., Ltd.
#         Product Name: A520M-A PRO (MS-7C96)
#         Version: 1.0
#         Serial Number: 07C9611_O11B469063
#         Asset Tag: To be filled by O.E.M.
#         Features:
#                 Board is a hosting board
#                 Board is replaceable
#         Location In Chassis: To be filled by O.E.M.
#         Chassis Handle: 0x0003
#         Type: Motherboard
#         Contained Object Handles: 0
# Handle 0x0037, DMI type 41, 11 bytes
# Onboard Device
#         Reference Designation: Realtek ALC1220 # 板载声卡
#         Type: Sound
#         Status: Enabled
#         Type Instance: 1
#         Bus Address: 0000:25:00.4
sudo lshw -C system
# mycomputer-gtx1065
#     description: Desktop Computer
#     product: MS-7C96 (To be filled by O.E.M.)
#     vendor: Micro-Star International Co., Ltd.
#     version: 1.0
#     serial: To be filled by O.E.M.
#     width: 64 bits
#     capabilities: smbios-2.8 dmi-2.8 smp vsyscall32
cat /sys/class/dmi/id/board_name
# A520M-A PRO (MS-7C96)
```

## 内存
```sh
free -h

sudo dmidecode -t memory
# Handle 0x0011, DMI type 16, 23 bytes
# Physical Memory Array
#         Location: System Board Or Motherboard
#         Use: System Memory
#         Error Correction Type: None
#         Maximum Capacity: 128 GB
#         Error Information Handle: 0x0010
#         Number Of Devices: 2
# Handle 0x0019, DMI type 17, 40 bytes
# Memory Device
#         Array Handle: 0x0011
#         Error Information Handle: 0x0018
#         Total Width: 64 bits
#         Data Width: 64 bits
#         Size: 8 GB
#         Form Factor: DIMM
#         Set: None
#         Locator: DIMM 0
#         Bank Locator: P0 CHANNEL A
#         Type: DDR4
#         Type Detail: Synchronous Unbuffered (Unregistered)
#         Speed: 3200 MT/s
#         Manufacturer: Unknown
#         Serial Number: D4083222
#         Asset Tag: Not Specified
#         Part Number: VGM4UX32C188G-STACRN
#         Rank: 1
#         Configured Memory Speed: 3200 MT/s  # 需要开启A-XMP
#         Minimum Voltage: 1.2 V
#         Maximum Voltage: 1.2 V
#         Configured Voltage: 1.2 V
# Handle 0x001C, DMI type 17, 40 bytes  # 2 * 8GB 内存条
#         Bank Locator: P0 CHANNEL B    # 双通道

sudo apt install i2c-tools
sudo decode-dimms # 可能需要 sudo modprobe ee1004
# Decoding EEPROM: /sys/bus/i2c/drivers/ee1004/0-0050
# Guessing DIMM is in                              bank 1
# Kernel driver used                               ee1004
# ---=== SPD EEPROM Information ===---
# EEPROM CRC of bytes 0-125                        OK (0xBD97)
# # of bytes written to SDRAM EEPROM               384
# Total number of bytes in EEPROM                  512
# Fundamental Memory type                          DDR4 SDRAM
# SPD Revision                                     1.1
# Module Type                                      UDIMM
# EEPROM CRC of bytes 128-253                      OK (0xB9C3)
# ---=== Memory Characteristics ===---
# Maximum module speed                             2400 MT/s (PC4-19200)
# Size                                             8192 MB
# Banks x Rows x Columns x Bits                    16 x 16 x 10 x 64
# SDRAM Device Width                               8 bits
# Ranks                                            1
# Primary Bus Width                                64 bits
# AA-RCD-RP-RAS (cycles)                           17-17-17-39
# Supported CAS Latencies                          18T, 17T, 16T, 15T, 14T, 13T, 12T, 11T, 10T
# ---=== Timings at Standard Speeds ===---
# AA-RCD-RP-RAS (cycles) as DDR4-2400              17-17-17-39
# AA-RCD-RP-RAS (cycles) as DDR4-2133              15-15-15-35
# AA-RCD-RP-RAS (cycles) as DDR4-1866              13-13-13-30
# AA-RCD-RP-RAS (cycles) as DDR4-1600              11-11-11-26
# ---=== Timing Parameters ===---
# Minimum Cycle Time (tCKmin)                      0.833 ns
# Maximum Cycle Time (tCKmax)                      1.600 ns
# Minimum CAS Latency Time (tAA)                   13.750 ns
# Minimum RAS to CAS Delay (tRCD)                  13.750 ns
# Minimum Row Precharge Delay (tRP)                13.750 ns
# Minimum Active to Precharge Delay (tRAS)         32.000 ns
# Minimum Active to Auto-Refresh Delay (tRC)       45.750 ns
# Minimum Recovery Delay (tRFC1)                   350.000 ns
# Minimum Recovery Delay (tRFC2)                   260.000 ns
# Minimum Recovery Delay (tRFC4)                   160.000 ns
# Minimum Four Activate Window Delay (tFAW)        21.000 ns
# Minimum Row Active to Row Active Delay (tRRD_S)  3.300 ns
# Minimum Row Active to Row Active Delay (tRRD_L)  4.900 ns
# Minimum CAS to CAS Delay (tCCD_L)                5.000 ns
# Minimum Write Recovery Time (tWR)                15.000 ns
# Minimum Write to Read Time (tWTR_S)              2.500 ns
# Minimum Write to Read Time (tWTR_L)              7.500 ns
# ---=== Other Information ===---
# Package Type                                     Monolithic
# Maximum Activate Count (MAC)                     Unlimited
# Post Package Repair                              One row per bank group
# Soft PPR                                         Supported
# Module Nominal Voltage                           1.2 V
# Thermal Sensor                                   No
# ---=== Physical Characteristics ===---
# Module Height                                    32 mm
# Module Thickness                                 2 mm front, 2 mm back
# Module Reference Card                            A revision 2
# ---=== Manufacturer Data ===---
# Module Manufacturer                              Gloway International Co. Ltd # 光威
# DRAM Manufacturer                                CXMT (former Innotron Memory Co Ltd) # 长鑫存储
# Manufacturing Date                               2024-W32
# Assembly Serial Number                           0xD4083222
# Part Number                                      VGM4UX32C188G-STACRN

# Decoding EEPROM: /sys/bus/i2c/drivers/ee1004/0-0051
# Guessing DIMM is in                              bank 2
# Kernel driver used                               ee1004
# ---=== Manufacturer Data ===---
# Module Manufacturer                              Gloway International Co. Ltd # 光威
# DRAM Manufacturer                                Micron Technology # 美光
# Manufacturing Date                               2024-W02
# Assembly Serial Number                           0xD400033D
# Part Number                                      TAC4U3200E18081C

# Number of SDRAM DIMMs detected and decoded: 2
```

## 磁盘
```sh
lsblk -f # blkid
# nvme0n1
# ├─nvme0n1p1       vfat        FAT32          5F39-2858                                   1G     1% /boot/efi
# ├─nvme0n1p2       ext4        1.0            bdaf3ad9-d681-44c9-a4e7-5f8817bee337      1.4G    22% /boot
# └─nvme0n1p3       LVM2_member LVM2 001       NPotXR-UNEj-cEja-1KA3-h6bS-iqPZ-gPfaoC
#   └─ubuntu--vg-ubuntu--lv
#                   ext4        1.0            3a5f860c-a18f-4ec4-ade3-289146ee9412     25.9G    73% /

df -hT
# /dev/mapper/ubuntu--vg-ubuntu--lv ext4   115G   83G   26G  77% /
# /dev/nvme0n1p2                    ext4   2.0G  432M  1.4G  24% /boot
# /dev/nvme0n1p1                    vfat   1.1G  6.1M  1.1G   1% /boot/efi
sudo vgdisplay ubuntu-vg # pvdisplay / vgdisplay / lvdisplay  
#   VG Size               <116.19 GiB
#   PE Size               4.00 MiB
#   Total PE              29744
#   Alloc PE / Size       29744 / <116.19 GiB
#   Free  PE / Size       0 / 0

du -h . | sort -hr | head -n 30
```

## CPU
```sh
lscpu -p # 查看 CPU 物理拓扑
# CPU,Core,Socket,Node,,L1d,L1i,L2,L3

lscpu
# Architecture:                x86_64
#   CPU op-mode(s):            32-bit, 64-bit
#   Address sizes:             43 bits physical, 48 bits virtual
#   Byte Order:                Little Endian
# CPU(s):                      6
#   On-line CPU(s) list:       0-5
# Vendor ID:                   AuthenticAMD
#   Model name:                AMD Ryzen 5 3500X 6-Core Processor
#     CPU family:              23
#     Model:                   113 # 对应 AMD Zen 2 微架构（Matisse）
#     Thread(s) per core:      1
#     Core(s) per socket:      6
#     Socket(s):               1
#     Stepping:                0
#     Frequency boost:         enabled
#     CPU max MHz:             3600.0000 # 基础频率MAX
#     CPU min MHz:             2200.0000
# Virtualization features:
#   Virtualization:            AMD-V
# Caches (sum of all):
#   L1d:                       192 KiB (6 instances)
#   L1i:                       192 KiB (6 instances)
#   L2:                        3 MiB (6 instances)
#   L3:                        32 MiB (2 instances)
# NUMA:
#   NUMA node(s):              1
#   NUMA node0 CPU(s):         0-5

sudo dmidecode -t processor

cat /proc/cpuinfo 
# cpu family   : 23
# model        : 113
# model name      : AMD Ryzen 5 3500X 6-Core Processor
# microcode    : 0x8701030
# physical id     : 0
# siblings        : 6
# cpu cores       : 6
```

## GPU
```sh
## 通用方式：
lspci -v | grep -A 20 -E "VGA|3D"
# 23:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 5GB] (rev a1) (prog-if 00 [VGA controller])
#         Subsystem: Device 7377:0000
#         Flags: bus master, fast devsel, latency 0, IRQ 67, IOMMU group 16
#         Memory at fb000000 (32-bit, non-prefetchable) [size=16M]
#         Memory at 7fe0000000 (64-bit, prefetchable) [size=256M]
#         Memory at 7ff0000000 (64-bit, prefetchable) [size=32M]
#         I/O ports at e000 [size=128]
#         Expansion ROM at fc000000 [virtual] [disabled] [size=512K]
#         Capabilities: <access denied>
#         Kernel driver in use: nvidia
#         Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
# 23:00.1 Audio device: NVIDIA Corporation GP106 High Definition Audio Controller (rev a1)
#         Subsystem: Device 7377:0000
#         Flags: bus master, fast devsel, latency 0, IRQ 71, IOMMU group 16
#         Memory at fc080000 (32-bit, non-prefetchable) [size=16K]
#         Capabilities: <access denied>
#         Kernel driver in use: snd_hda_intel
#         Kernel modules: snd_hda_intel

## NVIDIA
watch -n 1 nvidia-smi
```


## USB Bus
```sh
lsusb -t
/:  Bus 04.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/4p, 10000M
/:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/4p, 480M
    |__ Port 3: Dev 2, If 0, Class=Human Interface Device, Driver=usbhid, 12M
    |__ Port 3: Dev 2, If 1, Class=Human Interface Device, Driver=usbhid, 12M
/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/3p, 10000M
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/9p, 480M
    |__ Port 5: Dev 3, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 5: Dev 3, If 1, Class=Wireless, Driver=btusb, 12M
    |__ Port 8: Dev 4, If 0, Class=Vendor Specific Class, Driver=r8188eu, 480M
```
但其实 Bus 04/03 是**同一个** USB 3.x 物理端口在逻辑上的双总线投影。
对于我这个主板（MSI MS-7C96 “`微星A520M-A PRO`”） [msi官网链接](https://www.msi.cn/Motherboard/A520M-A-PRO/Specification) 
- Bus 04/03 是主机背面的 4个 USB 3.2 type-A 接口（xhci_hcd/4p）
- Bus 02/01 是主机前面 支持 3个 USB 3.2 type-A 的插针（xhci_hcd/3p）、主机前面 支持 4个 USB 2.0 type-A 的插针 和 主机背面 2个 USB 2.0 type-A 接口（xhci_hcd/9-3p）。

每个物理接口都被 `xhci_hcd` 驱动映射成了两个逻辑总线（USB 3.x 和 USB 2.0），因此在 `lsusb -t` 输出中会看到两组根集线器(root_hub)和对应的端口(Port)。

> **xhci_hcd驱动** 会同时创建 SuperSpeed(USB 3.x) 和 High-Speed(USB 2.0)通道，映射到不同编号的虚拟总线上。

每个蓝色的 USB 3.x 物理接口，内部实际上是由两组独立的信号线组成的：
- **USB 2.0 引脚**（D+/D-），用来兼容旧设备，最高速率 480 Mbps。
- **USB 3.x 引脚**（SSTX+/SSTX-、SSRX+/SSRX-），用于 5 Gbps 或 10 Gbps 的高速传输设备。
插入设备时，根据设备类型和其能力，只会激活其中一组引脚：
- 插入**USB 2.0 设备**（鼠标、U盘、RTL8188EU 便宜网卡等）→ 仅连接 USB 2.0 总线。
- 插入**USB 3.x 设备**（高速移动硬盘等）→ 仅连接 USB 3.x 总线，此时 USB 2.0 总线对应的端口会在电气上断开。

这意味着，同一个物理端口的两组信号线不可能同时工作，但在逻辑上它们必须被OS(操作系统)作为两个独立的端口来管理。因此，xHCI 控制器会向上报告两个“**根集线器**”（一个 USB 2.0 根集线器，一个 USB 3.x 根集线器）。每个根集线器拥有自己的端口集合，并被映射到不同编号的总线Bus上。

### USB 无线网卡
```sh
sudo lsusb -v -d 0bda:8179
# Bus 001 Device 002: ID 0bda:8179 Realtek Semiconductor Corp. RTL8188EUS 802.11n Wireless Network Adapter
# Device Descriptor:
#   ...
#   bcdUSB               2.00
#   idVendor           0x0bda Realtek Semiconductor Corp.
#   idProduct          0x8179 RTL8188EUS 802.11n Wireless Network Adapter
#   ...

cat /proc/net/wireless
# Inter-| sta-|   Quality        |   Discarded packets               | Missed | WE
#  face | tus | link level noise |  nwid  crypt   frag  retry   misc | beacon | 22
# wlx04ecbb92a1a1: 0000   63.  -51.  -256.       0      0      0      0      0        0
```

## 网卡
```sh
sudo lspci -nnkv | grep -A 20 -E "Ethernet"
# 22:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
#         Subsystem: Micro-Star International Co., Ltd. [MSI] RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [1462:7c96]
#         Flags: bus master, fast devsel, latency 0, IRQ 35, IOMMU group 15
#         I/O ports at f000 [size=256]
#         Memory at fc604000 (64-bit, non-prefetchable) [size=4K]
#         Memory at fc600000 (64-bit, non-prefetchable) [size=16K]
#         Capabilities: [40] Power Management version 3
#         Capabilities: [50] MSI: Enable- Count=1/1 Maskable- 64bit+
#         Capabilities: [70] Express Endpoint, MSI 01
#         Capabilities: [b0] MSI-X: Enable+ Count=4 Masked-
#         Capabilities: [100] Advanced Error Reporting
#         Capabilities: [140] Virtual Channel
#         Capabilities: [160] Device Serial Number 01-00-00-00-68-4c-e0-00
#         Capabilities: [170] Latency Tolerance Reporting
#         Capabilities: [178] L1 PM Substates
#         Kernel driver in use: r8169
#         Kernel modules: r8169

sudo lspci -nnkv | grep -A 20 -E "Wi-Fi"
# 23:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX210/AX211/AX411 160MHz [8086:2725] (rev 1a)
#         Subsystem: Intel Corporation Wi-Fi 6 AX210 160MHz [8086:0024]
#         Flags: bus master, fast devsel, latency 0, IRQ 33, IOMMU group 15
#         Memory at fc500000 (64-bit, non-prefetchable) [size=16K]
#         Capabilities: [c8] Power Management version 3
#         Capabilities: [d0] MSI: Enable- Count=1/1 Maskable- 64bit+
#         Capabilities: [40] Express Endpoint, MSI 00
#         Capabilities: [80] MSI-X: Enable+ Count=16 Masked-
#         Capabilities: [100] Advanced Error Reporting
#         Capabilities: [14c] Latency Tolerance Reporting
#         Capabilities: [154] L1 PM Substates
#         Kernel driver in use: iwlwifi
#         Kernel modules: iwlwifi
```

## PCI
```sh
lspci
```

## BIOS
```sh
sudo dmidecode -s bios-version
# 1.K0
mokutil --sb-state
# SecureBoot disabled

## BIOS
# Secure Boot: [Enabled]->[Disabled]
# Restore after AC Power Loss: [Power Off]->[Power On]
# A-XMP: [Disabled]->[Enabled]
```

## 其他
```sh
cat /proc/asound/cards
#  0 [NVidia         ]: HDA-Intel - HDA NVidia
#                       HDA NVidia at 0xfc080000 irq 71
#  1 [Generic        ]: HDA-Intel - HD-Audio Generic
#                       HD-Audio Generic at 0xfc400000 irq 73

sensors
# nvme-pci-0100
# Adapter: PCI adapter
# Composite:    +33.9°C  (low  = -273.1°C, high = +74.8°C)
#                        (crit = +84.8°C)
# k10temp-pci-00c3
# Adapter: PCI adapter
# Tctl:         +32.1°C
# Tccd1:        +32.5°C
```

速览：
| 配件 | 规格 | 价格(元) |
| :--- | :--- | :--- |
| 主板 | MSI A520M-A PRO | 350 |
| CPU | AMD Ryzen 5 3500X + 塔散 | 300 + 50 |
| 显卡 | GTX 1060 5G | 500 |
| 内存 | 2*光威 DDR4 3200 8G (长鑫颗粒) | 200 |
| 固态 | SSD 128G | 80 |
| 电源 | 长城 额定300W | 100 |
| 无线网卡 | AX210 芯片 | 100 |
| 机箱 | Tt 启航者 F4 | 120 |
| 机箱风扇 | 利民 TL-C12B V2 工包 (x2) | 20 * 2 |

一手估计：1840