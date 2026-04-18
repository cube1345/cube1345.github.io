---
title: 在 FPGA 上搭建 RISC-V 架构 CPU：基于 LiteX + VexRiscv + Xilinx Artix-7 的流程记录
date: 2026-04-18 23:20:00
updated: 2026-04-18 23:20:00
categories:
  - FPGA
tags:
  - RISC-V
  - FPGA
  - LiteX
  - VexRiscv
  - Artix-7
description: 以 Artix-7 平台为例，梳理基于 LiteX 和 VexRiscv 搭建 RISC-V SoC 的技术栈选择、硬件结构、构建流程与基础验证方法。
toc: true
---

这篇文章整理了一个基于 LiteX 与 VexRiscv 的 Artix-7 FPGA 小型 RISC-V SoC 搭建过程。内容以流程说明为主，侧重技术栈选择、系统结构、构建步骤和基础验证方法。

> 说明：文中的引脚约束、时钟配置和烧录流程基于一套具体的 Artix-7 开发环境，迁移到其他板卡时需要按实际原理图和约束文件调整。
## 引言

在嵌入式系统学习和开发中，CPU 架构的理解一直是核心环节。传统的理解方式是通过软件模拟器运行指令，但这种方式缺乏硬件层面的直观性。现场可编程门阵列（FPGA）的出现，让开发者能够在硬件层面上实现一个真实的 CPU——这不仅能够加深对计算机体系结构的理解，还为 RISC-V 架构的学习和原型验证提供了绝佳的平台。

  

本文将详细介绍如何在 Xilinx Artix-7 FPGA（xc7a50tfgg484-1）上，使用开源的 LiteX 框架和 VexRiscv 软核，搭建一个完整的 RISC-V 系统。整个过程不需要购买昂贵的开发板，只需要一块通用的 Artix-7 开发板即可完成。

  

## 一、技术栈介绍

  

### 1.1 为什么选择 RISC-V？

  

RISC-V 是一个开源的指令集架构（ISA），相比于 ARM 和 x86，它具有以下优势：

  

- 完全开源，无授权费用

- 简洁高效，易于学习和实现

- 模块化设计，可根据需求灵活定制

- 社区活跃，生态不断完善

  

### 1.2 VexRiscv：纯 Scala 编写的 RISC-V CPU

  

VexRiscv 是一个用 Scala 编写的 RISC-V CPU 实现，它采用模块化设计，支持 RV32I、RV32IMC、RV32EC 等多种指令集变体。作为一个软核（Soft CPU），它可以通过 HDL 代码直接在 FPGA 上实现，无需特殊的 ASIC 工艺。

  

### 1.3 LiteX：开源的 SoC 构建框架

  

LiteX 是一个强大的开源框架，用于构建基于 RISC-V 的系统芯片（SoC）。它提供了以下核心功能：

  

- 自动生成 Wishbone 总线互联

- 集成常用的外设（UART、SPI、Timer 等）

- 支持多种 CPU 软核（VexRiscv、Rocket、PulseLAN）

- 自动生成 Verilog 代码并调用 FPGA 工具链

  

### 1.4 Migen：Python 编写的 HDL

  

Migen 是 LiteX 的基础，它允许开发者使用 Python 语法描述数字逻辑电路。相比于传统的 Verilog/VHDL，Migen 具有更高的抽象层次，能够显著提升开发效率。

  

### 1.5 Xilinx Artix-7：低成本 FPGA 的选择

  

Artix-7 是 Xilinx 7 系列 FPGA 中定位最低成本的产品，非常适合学习和原型开发。本项目使用的 xc7a50tfgg484-1 型号具有以下资源：

  

- 53200 个逻辑单元

- 160 个 DSP Slice

- 4.9Mb Block RAM

- 6 个时钟管理模块（CMT）

  

这些资源足以容纳一个完整的 RISC-V CPU 系统。

  

## 二、硬件设计

  

### 2.1 系统架构

  

整个系统的架构如下：

  

```

+-----------------------------------------------------------+

|                     Artix-7 FPGA                          |

|  +-------------+    +-------------+    +-------------+  |

|  |   VexRiscv  |----|   Wishbone  |----|   UART      |  |

|  |    CPU      |    |    Bus      |    |   (H18/H19) |  |

|  +-------------+    +-------------+    +-------------+  |

|         |                  |                               |

|  +-------------+    +-------------+    +-------------+  |

|  |    ROM      |    |    SRAM     |    |   LED/KEY   |  |

|  |   (32KB)   |    |   (16KB)    |    | (H15/H13)   |  |

|  +-------------+    +-------------+    +-------------+  |

|                                                          |

|  +-------------+                                       |

|  |  S7PLL     |----> 50MHz system clock               |

|  | (25MHz in) |                                       |

|  +-------------+                                       |

+-----------------------------------------------------------+

```

  

### 2.2 引脚约束配置

  

在 `my_soc.py` 文件中，我们定义了所有引脚的约束：

  

```python

_io = [

    # 25MHz 输入时钟 (K4)

    ("clk25", 0, Pins("K4"), IOStandard("LVCMOS33")),

  

    # 串口引脚 (H18: RX, H19: TX)

    ("serial", 0,

        Subsignal("tx", Pins("H19")),

        Subsignal("rx", Pins("H18")),

        IOStandard("LVCMOS33")

    ),

  

    # 用户 LED (H15)

    ("user_led", 0, Pins("H15"), IOStandard("LVCMOS33")),

    # 用户按键/复位 (H13)

    ("user_btn", 0, Pins("H13"), IOStandard("LVCMOS33")),

]

```

  

| 引脚 | 功能 | 说明 |

|------|------|------|

| K4 | CLK_25M | 25MHz 晶振输入 |

| H18 | UART_RX | 串口接收 |

| H19 | UART_TX | 串口发送 |

| H13 | RST_BTN | 复位按键（按下为低电平） |

| H15 | USER_LED | 用户指示灯 |

  

### 2.3 时钟与复位设计

  

时钟管理模块（`_CRG`）负责将 25MHz 的输入时钟通过内部的 PLL 倍频到 50MHz，作为 CPU 的系统时钟：

  

```python

class _CRG(Module):

    def __init__(self, platform, sys_clk_freq):

        self.clock_domains.cd_sys = ClockDomain()

        clk25 = platform.request("clk25")

        self.submodules.pll = pll = S7PLL(speedgrade=-1)

        pll.register_clkin(clk25, 25e6)

        pll.create_clkout(self.cd_sys, sys_clk_freq)  # 输出 50MHz

```

  

复位设计使用按键实现：

- 按下 H13 按键（低电平）→ 系统复位

- 松开按键 → 系统重新启动

- LED (H15) 跟随按键状态：按住按键时 LED 熄灭，松开后点亮

  

## 三、软件构建

  

### 3.1 构建流程概述

  

整个构建流程由 Python 脚本自动完成：

  

1. **LiteX 初始化**：配置 CPU、总线、外设

2. **Verilog 生成**：将 Python 描述转换为 Verilog 代码

3. **综合**：使用 Vivado 进行综合

4. **实现**：布局布线

5. **比特流生成**：生成用于烧录的 .bit 文件

  

### 3.2 核心代码解析

  

```python

class BaseSoC(SoCCore):

    def __init__(self, platform, **kwargs):

        sys_clk_freq = int(50e6)  # 50MHz CPU 时钟

        self.submodules.crg = _CRG(platform, sys_clk_freq)

  

        SoCCore.__init__(self, platform, sys_clk_freq,

            cpu_type="vexriscv",              # 使用 VexRiscv CPU

            integrated_rom_size=0x8000,       # 32KB ROM

            integrated_sram_size=0x4000,      # 16KB SRAM

            with_ctrl=False,                  # 禁用_ctrl模块避免冲突

            **kwargs)

```

  

### 3.3 执行构建

  

```bash

# 清理旧构建

rm -rf build_my_soc/

  

# 执行构建

python3 my_soc.py

```

  

构建过程会生成以下文件：

  

| 文件/目录 | 说明 |

|-----------|------|

| `build_my_soc/gateware/top.bit` | FPGA 比特流文件 |

| `build_my_soc/gateware/top.v` | 生成的 Verilog 代码 |

| `build_my_soc/gateware/top.xdc` | 引脚约束文件 |

| `build_my_soc/software/bios/bios.bin` | BIOS 固件 |

| `build_my_soc/csr.csv` | 外设寄存器映射 |

  

## 四、FPGA 烧录

  

### 4.1 烧录方式选择

  

FPGA 烧录有两种方式：

  

| 方式 | 存储位置 | 掉电后 | 适用场景 |

|------|----------|--------|----------|

| Program Device | FPGA 内部 SRAM | 丢失 | 开发调试 |

| Program Flash | 外部 SPI Flash | 保留 | 产品部署 |

  

### 4.2 临时烧录（开发调试用）

  

1. 打开 Vivado Hardware Manager

2. 连接硬件目标

3. 右键点击设备 → Program Device

4. 选择生成的 .bit 文件

5. 点击 Program

  

### 4.3 永久烧录（到 Flash）

  

1. 添加 Configuration Memory Device（选择 W25Q128 或 MT25QL128）

2. 选择 .mcs 文件

3. 勾选 Erase、Program、Verify

4. 点击 OK 开始烧录

  

## 五、串口验证

  

### 5.1 连接配置

  

使用 MobaXterm 或其他串口工具连接：

  

| 参数 | 值 |

|------|-----|

| 波特率 | 115200 |

| 数据位 | 8 |

| 停止位 | 1 |

| 校验 | None |

| 流控 | None |

  

### 5.2 预期输出

  

烧录成功后，重启 FPGA，在串口终端应该看到：

  

```

        LiteX BIOS

        BIOS built on Apr  6 2026

  

Boot sequence:

  

Booting from ROM...

Initializing SDRAM...

SDRAM initialized.

Memory test: OK

  

LiteX BIOS (serial port)

-----------------------------------------

```

  

输入 `help` 可查看可用命令。

  

## 六、功能验证与调试

  

### 6.1 LED 和按键测试

  

- 按住 H13 按键：LED (H15) 熄灭

- 松开按键：LED 点亮，系统复位重启

  

### 6.2 串口通信测试

  

在串口终端输入任意字符，应该能看到回显。这验证了：

  

- UART 通信正常

- CPU 正在执行指令

- 串口接线正确

  

### 6.3 常见问题排查

  

| 问题 | 可能原因 | 解决方法 |

|------|----------|----------|

| 串口无输出 | 波特率不对、接线错误 | 检查 115200，检查 TX/RX 接线 |

| LED 不亮 | 引脚配置错误 | 确认 H15 约束是否正确 |

| 构建失败 | DRC 报错 | 添加 DRC 警告忽略命令 |

  

## 七、进阶开发

  

### 7.1 C 语言程序开发

  

LiteX 支持在 RISC-V CPU 上运行 C 语言程序：

  

```c

// hello.c

#include <stdio.h>

  

int main() {

    printf("Hello from RISC-V on FPGA!\n");

    return 0;

}

```

  

编译后通过串口下载到开发板运行。

  

### 7.2 添加更多外设

  

可以轻松添加以下外设：

  

- SPI Flash：用于存储更大程序

- I2C：连接传感器

- PWM：控制电机

- 以太网：网络通信

  

### 7.3 定制 CPU 配置

  

VexRiscv 支持多种配置选项：

  

- 调整流水线级数

- 启用/禁用缓存

- 添加/Debug 接口

- 配置中断控制器

  

## 八、总结

  

通过本文的讲解，我们成功在 Xilinx Artix-7 FPGA 上搭建了一个完整的 RISC-V 系统。这个系统包含：

  

- ✅ VexRiscv 32 位 RISC-V CPU（运行在 50MHz）

- ✅ 32KB ROM + 16KB SRAM

- ✅ UART 串口通信（115200 波特率）

- ✅ 用户 LED 和复位按键

  

整个过程展示了如何利用开源工具（LiteX、Migen、VexRiscv）在低成本 FPGA 上实现一个真实的 CPU 系统。这为 RISC-V 架构的学习、嵌入式系统的开发、以及硬件加速器的原型验证提供了坚实的基础。

  

未来可以进一步探索的方向包括：添加更多外设、实现更复杂的 SoC 设计、或者尝试在 FPGA 上运行完整的 Linux 系统。

  

---

  

*本文基于 LiteX 框架和 Xilinx Vivado 2022.2 编写*

  

---
