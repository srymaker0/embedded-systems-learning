# 从上电启动理解 BootROM、GSL 和 BSP

Hi3516CV610 上电后，CPU 先执行芯片内部固化的 BootROM 程序。BootROM 从启动介质
取得后续引导内容，将执行权交给 GSL；GSL 准备 DDR，再装载并启动 U-Boot；
U-Boot 将 Linux 内核放入内存，准备启动参数，随后跳转到内核入口。Linux 建立进程、
内存和设备管理环境，用户态初始化程序再通过启动脚本运行 EOS。

这条启动链中的每一阶段都在解决一个具体依赖：后面的程序需要什么条件，前面的
程序就先把这些条件准备好。CPU 最初从哪里取指，由硬件复位机制决定；后续去哪里
执行，则由正在运行的程序按照镜像格式和启动协议决定。

下面沿 KUBEE-B 的 SPI NAND 普通启动路径展开，源码采用
`Hi3516CV610_SDK_V1.0.2.0`、U-Boot 2022.07 和 Linux 5.10.221。板级选择由
`device/board_config.mk` 指向的配置文件决定；涉及具体地址时，使用
[`kubee_b-spi_nand-256M-none.mk`](../../../../hisilicon/hi3516cv610/device/kubee_b-spi_nand-256M-none.mk)
及其启动配置。

## 阅读顺序

1. [程序、存储器与 BSP](#1-启动链中的程序和存储器)和[复位取指](#2-cpu-怎样执行第一条指令)：先弄清有哪些对象，以及启动的起点怎样建立。
2. [GSL 的运行条件](#3-gsl-怎样准备内存和运行环境)和[U-Boot 的装载](#4-gsl-怎样找到并启动-u-boot)：理解 SRAM、DDR、镜像位置和入口地址之间的关系。
3. [U-Boot 启动 Linux](#5-u-boot-怎样启动-linux)和[Linux 启动 EOS](#6-linux-怎样建立-eos-的运行环境)：沿已有源码连接引导程序、内核和用户态。
4. [故障定位](#7-沿启动阶段定位故障)和[项目练习](#8-在现有项目上验证启动机制)：用日志、源码和运行信息判断启动停在哪里。

## 1. 启动链中的程序和存储器

### 1.1 每个阶段都由 CPU 执行

Hi3516CV610 是一颗 SoC，芯片里集成了 ARM Cortex-A7 CPU 核、存储控制器、串口
和媒体处理单元等硬件。CPU 负责执行指令，控制器负责与 DDR、Flash 等部件交互。

BootROM、GSL、U-Boot 和 Linux 都包含供 CPU 执行的机器指令。它们执行的时机、
保存的位置和承担的工作不同。普通启动的主要交接关系如下，箭头表示前一阶段准备
完成后，让 CPU 开始执行后一阶段：

```text
硬件释放 CPU 复位
        ↓
BootROM        从芯片规定的启动来源取得后续引导内容
        ↓
GSL            初始化 DDR，装载并启动 U-Boot
        ↓
U-Boot         装载内核镜像，准备参数，进入 Linux
        ↓
Linux          建立内存管理、调度、驱动和进程环境
        ↓
init           组织用户态初始化，执行启动脚本
        ↓
EOS            在 Linux 提供的环境中运行应用逻辑
```

前几段引导代码完成交接后，通常就结束了自己的启动任务。Linux 启动 EOS 后仍然
管理整个系统：应用的系统调用、任务调度和设备中断都需要内核参与。

### 1.2 程序保存在什么地方

程序的长期保存位置与执行时使用的内存，要分别看待。设备断电后，Flash 中的固件
仍然保留，DDR 中的运行内容会丢失；下次上电时需要重新装载程序。

| 存储器 | 断电后的内容 | 在这条启动链中的作用 |
| --- | --- | --- |
| 芯片内部 ROM | 保留 | 保存厂商固化的 BootROM 程序 |
| 板上的 SPI NAND Flash | 保留 | 保存启动镜像、内核镜像和文件系统等内容 |
| 芯片内部 SRAM | 丢失 | 为早期启动提供可用内存，保存 GSL 的早期代码、栈和数据 |
| 板上的 DDR | 丢失 | 初始化后保存后续引导程序、内核、应用及其运行数据 |

SRAM 和 DDR 都是运行内存，但使用条件不同。芯片为启动阶段提供了可用的内部
SRAM；外部 DDR 则需要控制器配置和训练。这个差别决定了 GSL 为什么先在小块
SRAM 中运行，再把后面的程序放进 DDR。

SPI NAND 中的数据要经过 Flash 控制器读取，才能进入可执行的内存位置。因此，
“程序位于 Flash 的哪个偏移”和“CPU 从哪个地址执行”是两个不同的问题。

### 1.3 BootROM、GSL 和 U-Boot 分别是什么

**BootROM 是固化在芯片内部 ROM 中的启动程序。** 芯片厂商在制造芯片时就提供了
它。电路满足启动条件后，CPU 可以直接执行这段程序，即使板上的产品固件还没有
烧录。BootROM 根据芯片的启动配置选择数据来源，处理规定格式的启动内容；安全
启动和下载模式也有各自的处理路径。

**GSL 是这套海思 SDK 中的一级引导程序。** 它位于 BootROM 之后，承担 DDR
初始化、后续镜像处理和 U-Boot 装载等工作。启用安全启动时，GSL 还参与后续
启动内容的验证。海思资料对这一阶段的职责有明确说明。
[来源：Hi35xxVxxx 安全启动使用指南](https://docs.hisilicon.com/projects/Hi3516CV610/main/api-reference/安全子系统/Hi35xxVxxx%20安全启动使用指南/Hi35xxVxxx%20安全启动使用指南.html)

**U-Boot 是加载和启动操作系统的引导程序。** 它可以提供串口命令行、存储读写、
环境变量和镜像启动功能。本项目由它读取 Linux 镜像，并按 ARM Linux 启动协议
把执行权交给内核。

GSL 和 U-Boot 的源码在 SDK 中参与构建，生成的内容进入启动镜像。BootROM 随
芯片提供，普通固件烧录更新的是外部存储器中的启动内容。

### 1.4 GSL 为什么放在 BSP 目录下

BSP 是 Board Support Package，即板级支持包。它组织让系统在具体芯片和板子上
运行所需的软件与配置，包括早期启动、DDR 参数、板级初始化、内核配置、设备树
和驱动适配等。各厂商 SDK 的组织方式会有所不同。

GSL 直接处理早期硬件和 DDR 初始化，因此属于 BSP 管理的组件。当前 SDK 的
目录关系是：

```text
sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/
└── components/gsl/
    ├── boot/start.S       早期汇编入口
    ├── boot/main.c        初始化与下一阶段交接
    ├── boot/flash.c       读取后续启动内容
    └── linker.lds         代码和数据的链接布局
```

BSP 在这里描述软件的归属，实际启动阶段则由 BootROM、GSL、U-Boot 等程序承担。
`bsp/components/gsl` 因而表示“板级支持包中的 GSL 实现”。源码组织关系和前面的
启动顺序分别回答了组件由哪部分管理、CPU 接下来执行哪段代码。

## 2. CPU 怎样执行第一条指令

### 2.1 复位机制建立第一个执行位置

在已经运行的 C 程序里，一个函数可以调用另一个函数。上电时还没有这样的调用者，
所以第一个执行位置要由硬件建立。

电源、时钟和复位电路使芯片进入规定的初始状态。满足启动条件并释放 CPU 复位后，
CPU 从体系结构与芯片设计规定的复位入口取指，芯片内部的地址映射把这次访问送到
启动 ROM。随后，CPU 就开始执行 ROM 中保存的机器指令。

```text
电源和时钟满足启动要求
        ↓
硬件释放 CPU 复位，建立初始执行状态
        ↓
CPU 向复位入口发出取指访问
        ↓
地址映射将访问送到启动 ROM
        ↓
取出机器指令，解码并执行
```

地址映射是硬件对访问地址的分流规则。例如，访问一段地址会到达 ROM，访问另一段
地址会到达 SRAM 或外设寄存器。复位时用于启动的映射已经由芯片提供，后续软件
才能在这个基础上继续执行。

运行过程中，程序计数器 PC 描述指令执行的位置。普通指令通常继续顺序执行，
分支指令改变后续执行位置；流水线和预取等细节由 CPU 实现。这里最关键的关系是：
复位建立起点，取指得到代码，执行代码后再决定下一步去哪里。

### 2.2 BootROM 和 BOOTRAM 的地址

Hi3516CV610 用户指南给出了以下地址区域：

| CPU 地址范围 | 用途 |
| --- | --- |
| `0x00000000`～`0x00007fff` | 指向 BootROM 的 32 KB 地址区域 |
| `0x04000000`～`0x04007fff` | BootROM 的 32 KB 地址区域 |
| `0x04020000`～`0x04033fff` | 80 KB BOOTRAM，即启动阶段使用的内部 SRAM |

这些区域说明了 CPU 通过什么地址访问启动存储器。BootROM 处理后续镜像后，
执行权会进入 GSL；本项目 GSL 的早期入口位于 `0x04022600`，落在 BOOTRAM
范围内。[来源：Hi3516CV610 产品概述与地址映射](https://docs.hisilicon.com/projects/Hi3516CV610/main/overall-architecture/Hi3516CV610%20超高清智慧视觉%20SoC%20用户指南/01-1%20产品概述.html)

SDK 提供了 GSL 的入口汇编。BootROM 内部第一条指令的具体编码还需要对应的
厂商资料或硬件调试证据；后面逐指令阅读的起点是 GSL 的 `_start`。

### 2.3 启动模式决定 BootROM 从哪里取得后续内容

Hi3516CV610 支持 SPI NOR、SPI NAND、eMMC 等启动来源。芯片通过启动模式管脚
等硬件配置取得选择结果，BootROM 据此使用相应的读取方式，再按芯片约定的镜像
格式处理启动内容。管脚组合和采样条件由芯片手册规定，板上的连接见原理图。
[来源：Hi3516CV610 启动模式](https://docs.hisilicon.com/projects/Hi3516CV610/main/overall-architecture/Hi3516CV610%20超高清智慧视觉%20SoC%20用户指南/01-1%20产品概述.html)

板级配置中的 `BOOT_MEDIA=spi_nand` 则作用于宿主机构建：它让构建系统选择
SPI NAND 相关配置和产物。硬件负责选择实际启动来源，构建配置负责准备与之匹配
的内容，两者需要一致。

例如，板子按 SPI NAND 启动时，BootROM 会沿 NAND 读取路径寻找启动内容。
如果生成和烧录的却是另一种介质使用的布局，读取或镜像识别就可能在这一阶段失败。

## 3. GSL 怎样准备内存和运行环境

### 3.1 DDR 初始化解决什么问题

CPU 通过 DDR 控制器访问外部内存。控制器需要知道内存组织、工作参数和时序，
并完成初始化和训练。训练会根据实际电路条件校准相关时序，使数据在合适的时间
被发送和采样。

假设现在要把 U-Boot 放到 DDR 的某个地址，但 DDR 控制器还没有准备好。一次
写操作可能没有正确保存数据，随后 CPU 跳到这个地址，就读不到预期的机器指令。
启动链需要先有一段能在已有内存中执行的程序，解决这个依赖。

GSL 使用芯片内部 SRAM 开始运行，在 SRAM 中建立栈和早期数据，再配置并训练
DDR。DDR 可用后，它才继续处理需要放入 DDR 的后续代码。海思的 U-Boot 移植
指南也按这个关系说明 GSL 与 U-Boot 的分工。
[来源：Hi35xxVxxx U-Boot 移植应用开发指南](https://docs.hisilicon.com/projects/Hi3516CV610/main/api-reference/BSP/Hi35xxVxxx%20U-boot%20移植应用开发指南/Hi35xxVxxx%20U-boot%20移植应用开发指南.html)

这也是 BSP 工作与具体硬件联系紧密的原因。厂商提供启动代码后，板上的 DDR
组织和电路条件仍然需要与初始化参数匹配；早期读写失败时，必须沿这些依赖定位。

### 3.2 链接布局决定 GSL 的代码地址

[GSL 的链接脚本](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/components/gsl/linker.lds)
将入口符号指定为 `_start`，并为早期代码安排 SRAM 区域：

```ld
ENTRY(_start)

/* MEMORY 中的 SRAM 区域 */
sram (rx) : ORIGIN = 0x4022600, LENGTH = 0x9400
```

链接器把各目标文件中的代码、数据和符号安排到输出程序中，因此 `_start` 最终
对应一个具体地址。前一阶段按芯片启动约定将 GSL 的早期内容准备到 SRAM，再
交接到这个入口。

`ENTRY(_start)` 描述的是链接产物的入口；CPU 的复位入口由第 2 节的硬件机制
建立。硬件起点、镜像装载约定和链接布局各自承担一部分工作，共同决定代码能否
被正确执行。[来源：GNU ld Entry Point](https://sourceware.org/binutils/docs/ld/Entry-Point.html)

这份链接脚本还安排了后续放入 DDR 的代码和数据区域。GSL 早期使用 SRAM，
DDR 可用后再搬运相应内容，使初始化前后的代码能使用各自需要的内存。

### 3.3 从汇编入口到 C 函数

[GSL 的 `boot/start.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/components/gsl/boot/start.S)
从一条分支指令开始：

```asm
.globl _start
_start:
    b reset
```

`_start` 和 `reset` 是源码中的位置标记。汇编、链接完成后，跳转关系被编码为
机器指令；CPU 执行 `b` 时，根据编码确定目标地址。符号名帮助开发者阅读，运行
所需的地址关系则保存在程序内容中。

接下来的 `reset` 路径设置异常向量，并通过 `smc #0` 进入对应处理入口，再到达
`boot`。随后完成栈和 `.bss` 初始化，才调用 `main_entry()`。相关指令摘录如下，
保留与 C 运行环境有关的部分：

```asm
ldr r0, =STACK_ADDR
mov sp, r0

ldr r0, =__bss_start
mov r1, #0
ldr r2, =__bss_end
subs r2, r2, r0
bl clear_data

/* 设置后续使用的异常向量后 */
bl main_entry
```

C 函数可能用栈保存寄存器、局部对象和调用状态，所以栈指针必须先指向可用内存。
静态存储期对象的零初值也需要建立，清理 `.bss` 就承担这部分准备。GSL 执行时
Linux 尚未运行，这些条件由早期汇编代码自己完成。

进入 `main_entry()` 后，GSL 已经具备运行这段 C 代码所需的基本条件。它仍然
直接操作早期硬件，Linux 的进程、文件描述符和系统调用要到后面的内核阶段才建立。

### 3.4 在 `main_entry()` 中找到 DDR 与装载的先后关系

[GSL 的 `boot/main.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/components/gsl/boot/main.c)
在取得板级参数和启动数据来源后，执行下面这些关键步骤：

```text
main_entry()
    │
    ├── handle_ree_key_and_param(...)
    │       取得后续初始化需要的参数等内容
    │
    ├── system_init(...)
    │       configure_ddr_parameters(...)
    │       ddrtrn_start(...) 等初始化
    │
    ├── handle_uboot(...)
    │       读取并处理 U-Boot 描述信息和代码
    │
    ├── copy_gsl_ddr_code()
    ├── store_booting_param()
    └── load_and_start_uboot()
```

这里提取了普通芯片构建中与内存和装载有关的调用；安全检查、辅助初始化及 FPGA
等条件分支在完整函数中保留。

这个顺序解释了各步骤之间的依赖：板级参数参与硬件初始化，DDR 可用后才能存放
后续镜像；镜像读取和处理完成后，GSL 才能取得交接所需的内容和入口地址。

## 4. GSL 怎样找到并启动 U-Boot

### 4.1 启动镜像把代码和装载信息放在一起

设备运行前，宿主机构建系统先把 GSL 和 U-Boot 源码编译成二进制，再按芯片要求
组织启动镜像。镜像除了程序代码，还需要让加载器知道各部分的含义、大小和位置。

当前维护的
[`tee_gsl.mk`](../../../../hisilicon/hi3516cv610/sdk_overlay/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/make/tee_gsl.mk)
会构建 GSL，准备 `gsl.bin`、板级参数 `reg_info.bin` 等输入，并调用镜像工具。
工具配置中还能看到 U-Boot 文件和入口字段：

| 配置字段 | 配置值 | 用途 |
| --- | --- | --- |
| `GSL_code_file` | `./input/gsl.bin` | GSL 代码输入 |
| `boot_param_file0` | `./input/reg_info.bin` | 板级启动参数输入 |
| `uboot_code_file` | `input/u-boot-original.bin` | U-Boot 启动内容输入 |
| `uboot_entry_point` | `0x41700000` | GSL 交接到 U-Boot 启动内容时使用的入口 |

字段来自
[`quick_build_config.json`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/tools/pc/image_tool/oem/quick_build_config.json)。
工具按规定布局组织这些输入及描述信息，生成 `boot_image.bin`。BootROM 和 GSL
分别按各自负责的格式处理镜像，取得下一阶段需要的内容。

因此，一份启动镜像需要同时满足“代码能正确执行”和“前一阶段能正确找到、读取
并解释这些代码”两个条件。镜像布局发生变化时，打包工具和读取代码必须遵循相同
的格式约定。

### 4.2 Flash 偏移、加载地址和执行入口各指什么

读取镜像时会同时遇到几类地址：

| 概念 | 参与的动作 | 示例含义 |
| --- | --- | --- |
| Flash 偏移 | 从存储器读取内容 | 从 Flash 的 `F` 位置开始读 |
| 内存加载地址 | 保存读出的内容 | 把内容写入 DDR 地址 `A` |
| 执行入口 | 让 CPU 开始执行程序 | 完成准备后跳到地址 `E` |

如果镜像就是可以直接运行的一段代码，加载地址和入口可能相同。如果前面带有
镜像头，或内部保存了压缩内容，就需要先解析、解压或搬运，才能到达可执行入口。

```text
Flash 偏移 F
    │  存储控制器读取
    ▼
DDR 加载地址 A
    │  解析镜像，按格式完成解压或搬运
    ▼
程序位于要求的位置，参数和 CPU 状态已准备好
    │  执行跳转
    ▼
入口地址 E
```

这三个位置之间的关系由镜像格式和启动实现决定。读代码时先辨认每个地址参与的
动作，再判断它们为什么相同或不同。

### 4.3 GSL 从哪里取得读取长度和目标地址

在
[`boot/flash.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/components/gsl/boot/flash.c)
的 `get_uboot_code_from_flash()` 中，装载参数来自以下赋值：

```c
dst = uboot_info->uboot_entry_point;
offset = get_uboot_code_flash_offset();
size = uboot_info->code_area_len;
```

Flash 分支随后叠加所选镜像的偏移，再调用 `fmc_read(dst, size, offset)` 或
对应的读取实现。这个操作把 Flash 中 `offset` 开始的 `size` 字节写到内存
`dst`。在这一阶段，目标内存地址采用镜像描述中的 `uboot_entry_point`。

装载结束后，`store_booting_param()` 保存入口地址，`load_and_start_uboot()`
按相应启动状态执行交接。源码中既有函数指针调用路径，也有需要切换 CPU 状态的
路径；`load_and_start_uboot()` 根据可信执行环境（TEE）等安全配置选择具体分支。

CPU 能到达下一阶段，是因为 GSL 读取了镜像中的信息，并最终执行了跳转指令。
源文件和函数的名称在构建时帮助组织程序，设备运行时使用的是这些已经形成的
地址和机器指令。

### 4.4 U-Boot 为什么先后使用两个地址

镜像工具把 U-Boot 入口配置为 `0x41700000`，对应
[压缩启动部分的链接地址](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/cpu/armv7/hi3516cv610/hw_compressed/u-boot.lds)。
而
[`hi3516cv610_defconfig`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/configs/hi3516cv610_defconfig)
将 U-Boot 主体的 `CONFIG_SYS_TEXT_BASE` 配置为 `0x41800000`。

它们之间有一个解压阶段：

```text
GSL 跳到 0x41700000
        ↓
执行 U-Boot 压缩启动代码
        ↓
将 U-Boot 主体解压到 0x41800000
        ↓
跳到 U-Boot 主体入口
```

对应的
[`startup.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/cpu/armv7/hi3516cv610/hw_compressed/startup.c)
先调用 `hw_dec_decompress()`，随后把 `CONFIG_SYS_TEXT_BASE` 作为函数入口
执行。U-Boot 主体运行后还会按自身配置处理重定位等工作。

压缩镜像因此有两次不同的交接：GSL 先启动负责解压的代码，解压代码再启动
U-Boot 主体。看到多个地址时，应按这两个阶段分别理解。

## 5. U-Boot 怎样启动 Linux

### 5.1 从自身初始化进入自动启动

U-Boot 主体也需要先准备栈、运行数据和板级资源，然后才进入命令处理与自动启动。
ARM 入口可以从以下文件中的符号读起：

| 源码 | 阅读入口 |
| --- | --- |
| [`arch/arm/lib/vectors.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/lib/vectors.S) | `_start` 与复位分支 |
| [`arch/arm/cpu/armv7/start.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/cpu/armv7/start.S) | `reset` 与 `_main` 调用 |
| [`arch/arm/lib/crt0.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/lib/crt0.S) | `board_init_f()`、重定位和 `board_init_r()` |
| [`common/main.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/common/main.c) | `main_loop()` 与自动启动逻辑 |

在 ARM 常规启动路径中，这两个初始化函数以重定位为界：`board_init_f()` 先
完成早期准备，之后的代码搬运 U-Boot 并切换执行位置，`board_init_r()` 再组织
后续初始化。进入 `main_loop()` 后，U-Boot 根据环境和启动设置处理倒计时、
命令输入以及 `bootcmd`。串口上的 U-Boot 命令行就是这段引导软件提供的交互环境。

### 5.2 `bootcmd` 决定本次读取哪个系统镜像

板级配置选择了
[`nand_recovery_env.txt`](../../../../hisilicon/hi3516cv610/sdk_overlay/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/tools/pc/uboot_env/env_text/hi3516cv610/nand_recovery_env.txt)
作为环境模板。其中 `bootcmd` 根据启动状态选择普通系统或 recovery 路径；普通
路径中的 `boot_normal` 主要完成以下操作：

1. 从 NAND 读取内核镜像到内存。
2. 设置传给内核的启动参数。
3. 解析并启动内核镜像。

读取和启动使用的命令分别是：

```text
nand read 0x41000000 ${system_kernel_addr} ${system_kernel_size}
bootm 0x41000000
```

第一条命令中的 `system_kernel_addr` 和 `system_kernel_size` 指定 Flash 读取
范围，`0x41000000` 指定内存中的镜像位置。原始 `boot_normal` 在这两条命令
之间还会设置完整的 `bootargs`。

`bootcmd` 和 `boot_normal` 都是 U-Boot 解释执行的字符串。U-Boot 解析命令后
调用相应的存储、镜像和启动实现，由这些实现完成数据读取和 CPU 状态准备。

环境模板参与构建，设备还可以保存自己的环境值。分析具体设备的读取范围时，
需要同时看板级生成结果和设备实际使用的环境；分区与模板维护见
[产品配置与板级适配](../../../../hisilicon/hi3516cv610/docs/产品配置与板级适配.md)。

### 5.3 `bootm` 从镜像中取得执行入口

`bootm 0x41000000` 首先把参数解释为内存中的镜像位置。U-Boot 识别镜像格式、
处理镜像头和压缩内容，并准备后续执行所需的信息。

例如
[`boot/bootm.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/boot/bootm.c)
中的 legacy 镜像路径使用 `image_get_ep()` 取得镜像记录的入口，保存到
`images.ep`。其他封装由各自的解析路径处理。

到
[ARM Linux 交接代码](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/u-boot/u-boot-2022.07/arch/arm/lib/bootm.c)
时，代码把解析得到的入口转换为函数指针：

```c
kernel_entry = (void (*)(int, int, uint))images->ep;
```

准备好状态和参数后，普通调用路径执行：

```c
kernel_entry(0, machid, r2);
```

镜像位置与执行入口因而各有作用：前者帮助 U-Boot 找到需要解析的内容，后者
决定交接时开始执行的位置。带有内核自身解压代码的镜像，还会先经过那层入口，
再进入解压后的内核。

### 5.4 启动协议约定交接时的 CPU 状态

内核入口代码依赖约定好的寄存器和处理器状态。ARM 32 位 Linux 启动协议规定，
交接时 `r0` 为零，`r1` 按协议传递机器类型信息，`r2` 指向设备树或传统启动
参数；对 MMU、缓存和中断状态也有要求。
[来源：Linux 5.10 ARM 启动协议](https://docs.kernel.org/5.10/arm/booting.html)

这些约定使内核可以在尚未建立完整运行环境时，找到描述硬件和启动条件的数据。
U-Boot 的 ARM 交接代码会根据已经准备的内容，为 `r2` 选择设备树地址或传统
参数地址。

设备树描述硬件的组成和连接，例如设备寄存器地址、中断和总线关系。`bootargs`
则表达本次启动选择，例如 `console=ttyAMA0,115200` 指定内核控制台，`root=...`
指定根文件系统。内核开始运行后读取这些数据，继续建立系统。

## 6. Linux 怎样建立 EOS 的运行环境

### 6.1 早期汇编进入 `start_kernel()`

解压后 ARM 内核的入口位于
[`arch/arm/kernel/head.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/arch/arm/kernel/head.S)
中的 `stext`。这段汇编准备早期页表和地址空间切换所需的状态，再经过
[`head-common.S`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/arch/arm/kernel/head-common.S)
中的 `__mmap_switched` 等步骤，进入 `start_kernel()`。

[`init/main.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/init/main.c)
接着建立内存管理、调度、中断等设施。后续 `rest_init()` 创建将执行
`kernel_init()` 的 PID 1，继续完成初始化，并选择用户态初始化程序。

```text
内核入口 stext
        ↓
准备早期页表、地址空间切换和 C 运行条件
        ↓
start_kernel()
        ↓
建立内核基础设施
        ↓
rest_init() 创建 PID 1
        ↓
kernel_init() 继续初始化，准备根文件系统，执行用户态 init
```

EOS 使用的线程调度、虚拟内存、设备接口和文件系统，都依赖这些内核设施。因此，
从 U-Boot 跳入内核到运行 EOS 之间，还存在一段必要的系统初始化过程。

### 6.2 根文件系统把用户态程序提供给内核

内核镜像包含内核自己的代码，而 `/sbin/init`、Shell、动态库和启动脚本保存在
文件系统中。执行用户态程序前，内核必须能访问这些文件。

普通启动模板包含以下参数：

```text
root=ubi0:rootfs rootfstype=ubifs ro ubi.mtd=system
```

它们描述了 NAND 根文件系统的访问路径：

```text
名为 system 的 MTD 分区
        ↓  交给 UBI 管理
UBI 设备 ubi0
        ↓  找到名为 rootfs 的卷
UBIFS 根文件系统
        ↓
/sbin/init、/etc/inittab、/etc/init.d/rcS 等文件
```

MTD 提供原始 Flash 访问抽象，UBI 在其上管理卷并承担相应的 Flash 管理工作，
UBIFS 则在 UBI 卷上提供文件系统。`ubi.mtd=system` 指定交给 UBI 的分区，
`ubi0:rootfs` 按 UBI 设备编号和卷名选择根卷，`ro` 指定以只读方式挂载根。
[来源：Linux 5.10 UBIFS 文档](https://docs.kernel.org/5.10/filesystems/ubifs.html)

访问这条路径所需的存储驱动和文件系统支持必须在挂载前可用，通常由内核内建，
或通过合适的初始内存文件系统准备。如果唯一需要的驱动放在尚未能挂载的根卷里，
就会形成访问依赖，系统无法沿这条路径继续启动。
[来源：Linux init 执行失败排查](https://docs.kernel.org/5.10/admin-guide/init.html)

recovery 模板使用 `rdinit=/init`，由相应的初始内存文件系统提供初始化程序。
它与普通系统从 UBI 根卷启动的路径不同，日志分析应先确认本次选择的启动模式。

### 6.3 init 组织用户态初始化

`kernel_init()` 根据启动参数和配置选择用户态初始化程序，普通路径中可见
`/sbin/init` 等候选。本项目使用 BusyBox 体系的初始化配置，
[`inittab`](../../../../hisilicon/hi3516cv610/fs_overlay/rootfs/common/etc/inittab) 指定：

```text
::sysinit:/etc/init.d/rcS
```

init 据此运行 `/etc/init.d/rcS`。启用 autorun 配置时，
[`rcS`](../../../../hisilicon/hi3516cv610/fs_overlay/rootfs/autorun/etc/init.d/rcS)
先执行 `/etc/init.d/S[0-9][0-9]*` 中可执行的脚本，再执行
`/vendor/etc/init.d/S[0-9][0-9]*` 中可执行的脚本。

脚本的先后顺序承担实际依赖：先挂载产品卷，后面的程序才能从 `/vendor` 和
`/app` 找到模块、库、配置和可执行文件。此时已经进入 Linux 用户态，脚本通过
系统调用完成挂载、创建进程和执行程序等操作。

### 6.4 从产品卷到 `eos_sample`

固件和 EOS 仓库共同给出了应用启动路径：

```text
init 执行 rcS
        ↓
S20system-volumes 挂载 /vendor、/app 等产品卷
        ↓
S92autorun 选择应用启动方式
        ↓
/app/bin/sample_start.sh（由 OTA worker 或直接路径启动）
        ↓
设置库搜索路径，进入 /app/bin
        ↓
exec ./eos_sample
```

[`S20system-volumes`](../../../../hisilicon/hi3516cv610/fs_overlay/rootfs/common/etc/init.d/S20system-volumes)
组织产品卷挂载；
[`S92autorun`](../../../../hisilicon/hi3516cv610/fs_overlay/vendor/common/etc/init.d/S92autorun)
在 OTA 程序可执行时调用 `start-worker`，否则直接启动脚本。OTA 的
[`product.json`](../../../../hisilicon/hi3516cv610/ota/product.json)
将 `app_start` 配置为 `/app/bin/sample_start.sh`，由 worker 按启动状态管理。

EOS 仓库的
[KUBEE-B 启动脚本](../../../../app/eos/overlay/kubee_b/app/bin/sample_start.sh)
设置库搜索路径，进入应用目录，最后执行 `exec ./eos_sample`。这一步把前面的
芯片启动、内核初始化和产品文件系统连接到了现有应用。

排查具体设备时，应用是否启动还取决于已安装脚本、产品卷状态和 OTA worker 的
当前处理结果。相关 OTA 行为由
[OTA 设计](../../../../hisilicon/hi3516cv610/ota/docs/design/OTA设计.md)维护。

### 6.5 ELF 入口怎样到达应用的 `main()`

Shell 的 `exec` 请求内核用目标程序替换当前进程的程序映像。内核读取 ELF，
建立用户地址空间，并准备开始执行的用户态位置。

如果 ELF 指定动态解释器，内核先安排执行动态加载器，再由加载器和程序运行时
完成必要准备，最终进入应用的 `main()`；静态程序没有同样的动态解释器步骤。
对应装载逻辑可在
[`fs/binfmt_elf.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/fs/binfmt_elf.c)
中阅读。

到这里，启动过程已经从“在指定物理位置准备并执行引导代码”，走到了“由操作
系统创建用户地址空间并执行应用”。EOS 后续使用设备和系统资源时，由 Linux
继续提供调度、系统调用和驱动服务。

## 7. 沿启动阶段定位故障

启动失败时，首先找到最后一个已经有证据的阶段，再检查它交接给下一阶段所需的
条件。串口日志、错误信息和当前可用的交互环境，分别能提供不同的定位依据。

| 观察到的现象 | 已到达的位置 | 优先检查 |
| --- | --- | --- |
| 串口没有输出 | 需要先确认输出链路和启用输出的阶段 | 供电、时钟与复位、串口连接和参数、所用 UART、启动介质 |
| 有 GSL 日志，随后停止 | 产生最后一条日志的 GSL 路径 | 对应初始化步骤、DDR、镜像读取和验证结果 |
| 能进入 U-Boot 命令行 | U-Boot 命令处理阶段 | 环境变量、Flash 读取范围、镜像格式和参数 |
| 显示 `Starting kernel ...` 后无输出 | U-Boot 准备交接内核的位置 | 内核入口、传参、控制台与早期串口配置、内存布局 |
| 内核报告根文件系统挂载失败 | 内核已经执行到根文件系统准备阶段 | `root`、`ubi.mtd`、分区和卷名、驱动与文件系统支持 |
| 根文件系统可用，init 执行失败 | 用户态初始化程序的装载边界 | init 文件、权限、架构、解释器和依赖库 |
| Linux Shell 可用，EOS 未运行 | 内核和部分用户态环境已可用 | 产品卷、启动脚本、OTA 启动管理、应用和动态库 |

比如，U-Boot 输出 `Starting kernel ...` 之后没有日志，可以先核对传给内核的
控制台参数。若控制台配置错误，内核可能已经执行，只是输出没有出现在正在监听
的串口；若入口或内存布局错误，则可能在内核早期就停止。需要新的观察来区分
这两类原因。

同样，进入 U-Boot 说明 DDR 已支持走到当前代码位置，但内存的全范围稳定性还
需要独立验证。后续出现随机崩溃时，应结合错误地址、访问范围和复现条件判断，
避免把“曾经启动成功”当成所有内存访问都可靠的依据。

排查时保留从上电开始的完整日志，记录实际镜像、启动模式和参数。每次修改围绕
一个待验证原因展开，观察它是否改变了预期阶段，才能逐步建立因果关系。

## 8. 在现有项目上验证启动机制

宿主机练习从固件仓库根目录执行，读取板级配置、展开的 SDK 源码和已有产物。
SDK 准备方式见[环境与源码准备](../../../../hisilicon/hi3516cv610/docs/环境与源码准备.md)。`sdk_overlay` 保存长期维护
的覆盖文件，`sdk` 提供准备后的完整源码；正式修改时沿构建规则找到对应维护入口。

### 8.1 确认板型和启动配置

```bash
readlink device/board_config.mk
rg -n '^(export )?(CHIP|BOOT_MEDIA|KERNEL_DTS|UBOOT_ENV_TEMPLATE|CFG_SDK_AUTORUN)=' device/board_config.mk
rg -n '^(SDK_NAME|KERNEL_VERSION|U_BOOT_NAME|BUSYBOX_NAME)=' scripts/versions.conf
```

选择上述 KUBEE-B 配置时，可以找到 `BOOT_MEDIA=spi_nand`、
`KERNEL_DTS=kubee_b` 和 `UBOOT_ENV_TEMPLATE=nand_recovery_env.txt`。
它们分别说明构建采用的存储介质、设备树选择和 U-Boot 环境模板。

先沿 `UBOOT_ENV_TEMPLATE` 找到环境文件，再定位 `bootcmd` 和 `boot_normal`，
将其中的 NAND 读取地址、镜像内存地址和 `bootargs` 分别标出来。这个练习完成后，
应能说明构建配置怎样影响 U-Boot 读取和启动内核的行为。

### 8.2 将 GSL 源码入口与机器指令对应起来

在同一个宿主机 Shell 中设置源码目录变量并查找入口：

```bash
gsl_dir=sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source/bsp/components/gsl
rg -n 'ENTRY|ORIGIN' "$gsl_dir/linker.lds"
rg -n '^_start:|^reset:|^boot:|STACK_ADDR|clear_bss|main_entry' "$gsl_dir/boot/start.S"
```

链接脚本给出 `_start` 和 SRAM 起始地址，汇编代码给出分支、栈初始化和进入 C
函数的顺序。如果构建目录已有 `pub/gsl` 和 `pub/gsl.asm`，还可以查看 ELF 头
和反汇编；`readelf` 在宿主机解析文件，不执行 ARM 程序：

```bash
readelf -h "$gsl_dir/pub/gsl"
rg -n -A 3 '<_start>:' "$gsl_dir/pub/gsl.asm"
```

与这份链接布局匹配的 ELF 入口为 `0x4022600`，对应入口反汇编为：

```text
04022600 <_start>:
 4022600: ea000016    b 4022660 <reset>
```

这条指令展示了源码符号怎样变成地址关系：`_start` 位于 `0x04022600`，目标
`reset` 位于 `0x04022660`，机器指令 `ea000016` 编码了这一跳转。

如果继续检查编码，这是一条正向 ARM A32 分支。它使用的 PC 基准为指令地址加
8，立即数 `0x16` 左移两位得到偏移 `0x58`：

```text
0x04022600 + 8 + (0x16 << 2) = 0x04022660
```

这个计算对应眼前这条 A32 指令；Thumb 或其他跳转形式需要按各自编码解释。
没有已有产物时，可先完成源码与链接布局的对应，待常规构建产生文件后再观察。

### 8.3 追踪 GSL 的读取与交接

继续使用前一节的 `gsl_dir`：

```bash
rg -n 'system_init|handle_uboot|store_booting_param|load_and_start_uboot|ddrtrn_start' "$gsl_dir/boot/main.c"
rg -n 'uboot_entry_point|get_uboot_code_flash_offset|code_area_len|fmc_read' "$gsl_dir/boot/flash.c"
```

打开命中的函数，按依赖顺序记录：

1. DDR 参数在哪里取得，初始化在哪一步完成。
2. Flash 偏移、读取长度和目标内存地址分别来自哪里。
3. 入口地址在哪里保存，最终由哪条路径使用。
4. 进入 `0x41700000` 后，谁将程序解压到 `0x41800000`。

完成后用一段话解释：改变 U-Boot 入口地址时，为什么还需要核对实际代码布局、
内存范围和前后阶段的约定。能够从这些具体依赖解释结果，比记住两个地址更有用。

### 8.4 在运行中的设备上观察启动结果

下面的命令在目标板已有的 Linux Shell 中执行，读取当前运行信息：

```sh
cat /proc/cmdline
cat /proc/1/comm
readlink /proc/1/exe
cat /proc/mtd
cat /proc/mounts
```

按顺序核对以下关系：

1. 在 `/proc/cmdline` 中找到控制台、`root` 和 `ubi.mtd`，与所用环境模板比较。
2. 从 `/proc/1/comm` 和 `/proc/1/exe` 确认 PID 1 的程序；若采用上述 BusyBox
   初始化路径，可以观察到相应的名称或可执行文件。
3. 在 `/proc/mtd` 中找到 `system` 等分区，再用 `/proc/mounts` 确认根文件系统
   以及 `/vendor`、`/app` 的实际挂载情况。
4. 将这些结果与 `rcS`、`S20system-volumes` 和 `S92autorun` 连接起来，说明
   EOS 启动脚本为什么能在相应路径找到程序和库。

宿主机练习反映源码和构建布局，板端观察反映已安装系统的运行结果。两侧出现差异
时，继续核对镜像、设备环境和持久配置，先确定设备实际使用了哪套内容。

## 参考资料

- [Hi3516CV610 用户指南：产品概述](https://docs.hisilicon.com/projects/Hi3516CV610/main/overall-architecture/Hi3516CV610%20超高清智慧视觉%20SoC%20用户指南/01-1%20产品概述.html)：启动模式、BootROM 和内部存储器地址映射。
- [Hi3516CV610 用户指南：系统](https://docs.hisilicon.com/projects/Hi3516CV610/main/overall-architecture/Hi3516CV610%20超高清智慧视觉%20SoC%20用户指南/01-3%20系统.html)：时钟、复位和系统控制。
- [Hi35xxVxxx U-Boot 移植应用开发指南](https://docs.hisilicon.com/projects/Hi3516CV610/main/api-reference/BSP/Hi35xxVxxx%20U-boot%20移植应用开发指南/Hi35xxVxxx%20U-boot%20移植应用开发指南.html)：GSL、DDR 初始化与启动镜像生成。具体目录和实现按所用 SDK 版本对应。
- [Hi35xxVxxx 安全启动使用指南](https://docs.hisilicon.com/projects/Hi3516CV610/main/api-reference/安全子系统/Hi35xxVxxx%20安全启动使用指南/Hi35xxVxxx%20安全启动使用指南.html)：GSL 的阶段职责和安全启动链。
- [Linux 5.10 ARM 启动协议](https://docs.kernel.org/5.10/arm/booting.html)：引导程序进入内核时的寄存器和处理器状态。
- [Linux 5.10 UBIFS](https://docs.kernel.org/5.10/filesystems/ubifs.html)：MTD、UBI、UBIFS 及根卷参数。
- [Linux 5.10 init 执行失败排查](https://docs.kernel.org/5.10/admin-guide/init.html)：根文件系统、init 和依赖的故障边界。
- [GNU ld Entry Point](https://sourceware.org/binutils/docs/ld/Entry-Point.html)：链接入口的定义。

源码链接中的 `sdk` 路径需要完成 SDK 准备；配套仓库布局见
[学习仓库说明](../../../README.md#配套工作区)。相关版本见 [`scripts/versions.conf`](../../../../hisilicon/hi3516cv610/scripts/versions.conf)，
文件系统内容的维护方式见[文件系统与功能裁剪](../../../../hisilicon/hi3516cv610/docs/文件系统与功能裁剪.md)。
