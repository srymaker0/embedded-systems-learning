# embedded-systems-learning

嵌入式 Linux 系统学习文档，通过原理解释、源码阅读和运行观察理解系统机制。

## 学习入口

| 主题 | 文档 | 学习内容 |
| --- | --- | --- |
| BSP 与系统启动 | [从上电启动理解 BootROM、GSL 和 BSP](docs/bsp/boot/启动链与BootROM和GSL.md) | 复位取指、存储器、GSL、U-Boot、Linux 与用户态程序的启动关系，以及源码阅读与运行观察 |

## 文档目录

```text
docs/
└── bsp/
    └── boot/
        └── 启动链与BootROM和GSL.md
```

## 配套工作区

源码练习需要对应版本的源码目录，命令按各节标注的宿主机或目标板环境执行。
本地源码链接依赖配套目录；GitHub 页面提供教程正文和公开参考资料。
