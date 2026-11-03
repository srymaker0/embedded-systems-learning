# embedded-systems-learning

嵌入式 Linux 系统学习文档，结合 Hi3516CV610 固件和 EOS 应用，沿实际启动过程、
源码调用和运行现象理解系统机制。

## 学习入口

| 主题 | 文档 | 学习内容 |
| --- | --- | --- |
| BSP 与系统启动 | [从上电启动理解 BootROM、GSL 和 BSP](docs/bsp/boot/启动链与BootROM和GSL.md) | 复位取指、存储器、GSL、U-Boot、Linux 和 EOS 的启动关系，以及源码阅读与运行观察 |

## 文档目录

```text
docs/
└── bsp/
    └── boot/
        └── 启动链与BootROM和GSL.md
```

## 配套工作区

正文中的源码链接用于本地阅读，按以下布局连接配套仓库：

```text
workspace/
├── embedded-systems-learning/
├── hisilicon/
│   └── hi3516cv610/
└── app/
    └── eos/
```

固件仓库中的 `sdk` 源码需要按其准备流程展开。教程的宿主机命令从固件仓库根目录
执行，板端命令在对应设备的 Linux Shell 中执行。

GitHub 页面可直接阅读教程和官方资料链接；跨仓源码链接依赖上面的本地目录布局，
在本地 Markdown 阅读器中使用。
