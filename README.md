# embedded-systems-learning

嵌入式 Linux 系统学习文档，通过原理解释、源码阅读和运行观察理解系统机制。

## 学习入口

| 主题 | 文档 | 学习内容 |
| --- | --- | --- |
| 启动代码阅读基础 | [从 C 代码到系统启动：读懂汇编、地址与运行环境](docs/bsp/boot/从C代码到系统启动.md) | 寄存器、函数调用、栈、链接布局、异常入口、重定位，以及内核和应用的运行环境 |
| BSP 与系统启动 | [从上电启动理解 BootROM、GSL 和 BSP](docs/bsp/boot/启动链与BootROM和GSL.md) | 复位取指、存储器、GSL、U-Boot、Linux 与用户态程序的启动关系，以及源码阅读与运行观察 |
| 内核并发与对象回收 | [从 AI 取帧故障理解中断、自旋锁、RCU 和文件回收](docs/linux/concurrency/AI取帧故障排查与内核机制详解.md) | 音频中断、自旋锁、RCU 宽限期、文件对象积压、OOM 与故障定位 |
| 文件缓存与内存分配 | [从 OTA 显示初始化失败理解文件缓存、内存回收与连续内存](docs/linux/memory/OTA显示初始化故障与Linux内存机制.md) | Page Cache 生命周期、脏页回写、伙伴系统、内存整理、DMA 分配与逐文件缓存管理 |

## 文档目录

```text
docs/
├── bsp/
│   └── boot/
│       ├── 从C代码到系统启动.md
│       └── 启动链与BootROM和GSL.md
└── linux/
    ├── concurrency/
    │   └── AI取帧故障排查与内核机制详解.md
    └── memory/
        └── OTA显示初始化故障与Linux内存机制.md
```

## 配套工作区

源码练习需要对应版本的源码目录，命令按各节标注的宿主机或目标板环境执行。
本地源码链接依赖配套目录；GitHub 页面提供教程正文和公开参考资料。
