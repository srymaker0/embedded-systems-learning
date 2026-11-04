# 从 AI 取帧故障理解中断、自旋锁、RCU 和文件回收

2026 年 10 月 27 日，在 Hi3516CV610 设备上排查到一个音频驱动缺陷：AI 取帧走到
特定错误分支时，没有释放自旋锁。之后音频中断再次申请这把锁，一个 CPU 卡在中断
处理路径；RCU 的回收进展受到影响，文件对象持续积压，最终可能触发 OOM。
海思 FAE 指出，normal 模式下高负载影响 `ot_ai_get` 线程调度，可能使取帧遇到
空队列，触发这个驱动缺陷。

这里的 **AI 是 Audio Input，即音频输入**；AENC 是音频编码，ADEC 是音频解码。
`ot_ai.ko` 是音频输入内核模块。文中的机制以 Linux 5.10.221、海思 SDK V1.0.2.0
B051 及当时的内核配置为例。当前构建已使用海思 P003 原始文件；下文的 Thumb 补丁
展示当时的临时修复。

## 阅读指南

可以按以下顺序阅读，也可以从关心的问题进入：

1. [故障全貌](#1-先看全貌一条音频故障为什么会影响内存)：音频卡住与文件对象积压怎样连在一起。
2. [音频中断](#2-从音频帧到中断谁在执行什么)和[自旋锁](#3-自旋锁先看为什么需要锁再看锁为什么卡住)：谁发起中断，哪条路径漏了解锁。
3. [RCU 机制、接口与示例](#4-rcu为什么对象已经不用了还不能马上释放)：宽限期何时结束，CPU 卡住后为什么不能回收。
4. [文件对象和 filp](#5-fdstruct-filefilp为什么-close-后内存还能增长)：`close()` 后 FD 与内核对象的生命周期。
5. [读 OOM 日志](#7-拿到-oom-日志应该按什么顺序读)和[排查步骤](#8-按普通工程方法一步一步把范围缩小)：哪些现象需要哪些证据。
6. [反汇编](#9-没有原厂-c-源码怎样读懂反汇编)、[Thumb 修复](#10-为什么修改-thumb-指令就能修复)和[验证](#11-修复验证)：如何检查没有源码的驱动。
7. [ref](#ref)：现场证据、外部资料及匹配源码。

## 1. 先看全貌：一条音频故障为什么会影响内存

把现场分成三条彼此连接的线，就能看出各部分各自做了什么：

```text
音频数据：  音频硬件 → DMA 写内存 → AI 输入驱动 → AENC → 应用读取编码数据
                          │
                          └─ 硬件通过中断通知 CPU 处理音频状态

执行进度：  取帧出错时漏解锁 → 音频中断等待这把锁 → 一个 CPU 无法正常退出中断

内存回收：  其他路径继续 open/close → struct file 等待 RCU 回收
                                      ↑
                          卡住的 CPU 无法提供 RCU 所需进展
                                      ↓
                           filp 对象积压，内存压力增长
```

音频的锁和文件对象并不需要互相调用：它们通过 **CPU 能否继续执行** 产生联系。
卡住的 CPU 影响全局回收条件，而另一个 CPU 仍可能创建和关闭文件。系统因而可以先
出现音频或显示异常，过一段时间才因内存耗尽触发 OOM。

## 2. 从音频帧到中断：谁在执行什么

### 2.1 线程调用驱动，硬件也能主动通知 CPU

应用进程由一个或多个线程执行。线程运行在用户态时，不能直接操作内核对象；
它通过系统调用进入内核，请求设备驱动完成操作。比如 `ioctl` 用于向设备提交
控制请求，`select` 用于等待文件描述符变为可读等状态。

音频硬件则不必等应用主动问它。麦克风采样后，音频接口按配置接收数据；DMA
（直接内存访问）可以将数据搬到内存，CPU 不必逐个样本搬运。当音频设备需要 CPU
处理状态时，硬件向中断控制器发出中断请求；控制器再把请求送给 CPU。CPU 暂停
当前执行流，进入驱动的中断处理程序，处理完后再返回被打断的位置。

```text
一条线程：  用户态发起 ioctl → 内核 AI/AIE 取帧处理 → 返回结果

硬件事件：  音频设备发中断 → GIC 中断控制器 → CPU 进入硬中断
                           → aio_interrupt_route → ai_drv_isr
                           → ai_dma_callback → 处理共享音频状态
```

这两条路径可能在不同时间或不同 CPU 上碰到同一份设备数据，所以驱动需要同步。

这里的“音频中断”与“硬件中断”描述的是同一个事件的不同方面：前者说它来自
音频设备，后者说它是由设备硬件发给 CPU 的中断。本次 `/proc/interrupts` 的一份
采样记录有下面一行，两个数字列依次是 CPU0、CPU1 的累计处理次数：

```text
 60:     112859          0     GIC-0  95 Level     AIO interrupt
```

其中 `AIO interrupt` 是音频设备中断的名称，`GIC-0` 是硬件中断控制器，`Level`
表示这条中断使用电平触发。那次采样中它主要由 CPU0 处理。设备树为 `aiao` 节点
声明了 `AIO` 中断；跨核原始栈和反汇编给出了从 `gic_handle_irq` 到
`aio_interrupt_route`、`ai_drv_isr` 和 `ai_dma_callback` 的处理路径。
IRQ 机制和设备树见文末参考资料 [[6]](#ref) [[16]](#ref)。

中断处理函数由软件编写并由 CPU 执行，但这不改变中断由硬件发起的性质。相比之下，
`ioctl` 是线程主动执行的系统调用，在本平台通过 ARM 的 SVC 指令陷入内核；这种
同步异常有时也被称为软件中断。Linux 的 softirq（软中断）则是内核安排的一类延后
处理机制，它和系统调用陷入以及外设硬件中断是不同概念。

这里导致卡住的是音频硬件中断中的驱动处理路径，`ai_dma_callback` 在硬中断
处理过程中执行。

### 2.2 这里的“帧”和“取帧超时”分别指什么

音频帧是一小段按顺序组织的采样数据。驱动维护缓冲区和相应状态，上游产生数据，
下游按接口取走并归还资源。

```text
上游准备好数据     →     驱动维护可用状态     →     下游等待并读取
       一段数据到达                 状态更新                    处理或编码
```

本次 EHAL 的 AENC 接收线程等待编码数据时，`select()` 返回 0 会打印
`get aenc stream select time out`，表示 **等待期限内没有等到可读事件**。
设备未开始产帧、上游停顿或应用被暂停时都可能出现这一现象。

### 2.3 两个 CPU 为什么能看到不同现象

CPU0 和 CPU1 是同一 SoC 上的两个处理器核，运行同一个 Linux，能访问共享内核
内存，但各自有执行进度，也可能处理不同中断。如果 CPU0 困在中断中，CPU1 仍可能
运行某些线程；“屏幕停住”与“另一个线程还在写日志”可以同时成立。

本次现场音频、UART、EDMA 中断主要落在 CPU0，因此 CPU0 停住会影响依赖这些中断
的功能。其他轮次也曾在 CPU1 抓到异常返回。受控实验把取帧线程和观察程序分到
不同 CPU，以保留观察通道。

### 2.4 normal 模式下高负载怎样触发故障

这里的 normal 是海思 AI 音频输入通道的工作模式 `OT_AI_CHN_MODE_NORMAL`。
该模式是 AI 通道的默认模式，启用 VQE 音频处理时，由 AI 内部线程完成处理。
本次板端采样显示 AI 设备 0、通道 0 的模式为 `normal`，VQE 已启用；进程中有
海思 SDK 的 `ot_ai_get` 线程。海思文档对工作模式和线程的说明见文末
参考资料 [[24]](#ref) [[25]](#ref)。

应用侧还有一条 EHAL 音频桥接线程：它调用 `hal_ai_get_chn_frame()`，再调用
`ss_mpi_ai_get_frame()` 获取音频帧，随后将帧送给 AENC 编码。它与 SDK 内部的
`ot_ai_get` 是两条线程：一次线程快照中，前者的线程号为 2202，后者为 2201。
EHAL 的调用路径见文末参考资料 [[17]](#ref) [[26]](#ref)。

```text
音频输入 → ot_ai_get：SDK 内部处理 → 等待应用读取的音频帧
                                                ↓
EHAL 桥接线程 → ss_mpi_ai_get_frame() → 取出帧 → AENC 编码
```

海思 FAE 给出的触发过程是：系统负载高时，`ot_ai_get` 线程的调度受到影响，
音频帧处理和应用取帧的时序发生变化，使 `ss_mpi_ai_get_frame()` 低概率遇到
取帧为空 [[28]](#ref)。若这次取帧进入 B051 驱动**加锁后发现队列为空**的错误
出口，就会漏掉解锁，随后音频硬件中断等待同一把锁，导致 CPU 卡住。

取帧为空表示这次没有取到可用帧。正常情况下，接口返回
`OT_ERR_AI_BUF_EMPTY`（`0xa0158016`）时，EHAL 将其转为超时并继续取帧
[[27]](#ref)。加锁前发现队列为空可以正常返回；加锁后发现队列为空，驱动必须
先解锁再返回。B051 的缺陷出在后者，具体控制流见第 3.5 节。

## 3. 自旋锁：先看为什么需要锁，再看锁为什么卡住

### 3.1 没有同步时，两条执行流可能读到半更新状态

假设驱动里有一个待处理帧队列。取走一帧通常不止改一个值：还要修改队列指针、
计数和帧状态。如果线程修改到一半时，另一路中断也来访问这些数据，就可能观察到
彼此不一致的状态。例如下面是为了说明竞态而写的简化过程：

```text
初始：队列里有帧 A，计数 = 1

取帧线程                           音频中断
读取到队列头为 A
准备摘除 A
                                   也读到 A 和计数 1
摘除 A，把计数改为 0
                                   按旧状态继续操作 A → 状态可能被破坏
```

需要作为整体完成的一组操作叫 **临界区**。所有访问这份共享状态的路径先取得
同一把锁，就能避免它们同时修改临界区。

### 3.2 自旋锁与互斥锁有什么不同

当一条执行流已持锁、另一条执行流也要拿锁时，后者需要等待。两类锁最直观的区别
在于等待方式：

| | 自旋锁 spinlock | 互斥锁 mutex |
| --- | --- | --- |
| 拿不到锁时 | 留在加锁路径等待，不通过普通任务睡眠等待锁 | 等待的线程可以睡眠，调度器可运行其他任务 |
| 常见使用场景 | 短临界区、线程与硬中断共享状态 | 可睡眠的线程上下文、等待可能较久的操作 |
| 能否让硬中断在锁上睡眠 | 不能 | 不能；硬中断上下文不适合使用普通 mutex |

硬中断处理不是一个可以像普通业务线程那样睡眠等待、再由调度器唤醒的独立任务。
本例 `ai_dma_callback` 位于硬中断调用链，所以不能直接把它使用的自旋锁替换为
mutex。自旋锁的临界区也不能做可能睡眠的操作；若要改成线程化中断和可睡眠的锁，
必须重新检查整条处理链，而不是只替换一个函数名。
Linux 自旋锁与 mutex 的使用约束见文末参考资料 [[4]](#ref) [[5]](#ref)。

本平台 ARM 自旋锁使用 ticket 排队算法；等待循环会执行 `wfe`，以减少无意义的
空转。它仍留在加锁路径，不能像 mutex 等待者那样通过普通任务睡眠让出执行流。
具体实现可看匹配内核的 `arch/arm/include/asm/spinlock.h`。

### 3.3 为什么取得锁之前还要保存并关闭本地中断

只考虑跨 CPU 还不够。假设线程和硬中断会拿同一把锁，且它们在同一个 CPU 上：

```text
CPU0 上的线程                         CPU0 上的音频中断
取得锁，进入临界区
被本地音频中断打断  ───────────────→ 进入中断处理
                                      尝试取得同一把锁，等待线程释放
只有中断返回后才能继续解锁  ←──────── 中断只有拿到锁才能继续并返回
```

这时线程等待中断返回，中断等待线程解锁，双方都无法继续。

`spin_lock_irqsave(&lock, flags)` 先保存 **当前 CPU** 的 IRQ 状态并关闭本地 IRQ，
再取得锁；这样本地硬中断不会在持锁期间打断它。另一个 CPU 的中断并未被这一步
关闭，因此仍要靠同一把自旋锁处理跨核运行。

`spin_unlock_irqrestore(&lock, flags)` 与之配对：释放锁，再恢复进入时保存的本地
IRQ 状态。如果调用前 IRQ 已关闭，恢复后也应保持关闭，不能无条件打开。锁和 IRQ
状态是两份不同的资源：只释放其中一份，不能代替另一份。

### 3.4 取号排队：本次自旋锁的内部状态怎么读

ticket 自旋锁可近似理解为叫号系统：`next` 是下一个要发出的号码，`owner` 是当前
轮到的号码。申请锁时先取号，等到 `owner` 轮到自己的号才进入；解锁时推进 `owner`，
再通知等待者。以下是简化示意，数字按十六进制写：

```text
锁空闲：    owner = b682，next = b682
线程取号：  owner = b682，next = b683   线程拿到 b682，持锁
中断取号：  owner = b682，next = b684   中断拿到 b683，等待
线程解锁：  owner = b683，next = b684   中断才轮到 b683
```

`owner != next` 表示锁处于使用状态。第 8.5 节的现场记录先显示错误返回时锁仍被
持有，随后 `next` 增加而 `owner` 长时间不前进。

### 3.5 本次的错误出口如何造成漏解锁

OSAL 是厂商对操作系统接口的封装。这里外层信号量保护一层操作，内部又使用自旋锁
和 IRQ 保存状态；两者不是同一把锁。释放外层信号量，不会替内部自旋锁解锁。

下面根据 B051 对象文件反汇编还原了与缺陷有关的控制流，省略参数检查和正常处理：

```c
down(&dev->sem);                         /* 取得外层信号量 */
if (query_busy_num(queue) == 0) {
    ret = NO_FRAME;
    goto out_sem;                       /* 还没有取得自旋锁 */
}

spin_lock_irqsave(&dev->lock, flags);     /* 此后同时持锁、保存了 IRQ 状态 */
if (queue_is_empty(queue)) {
    ret = NO_FRAME;
    goto out_sem;                       /* 缺陷：绕过 out_unlock */
}
/* 此处省略正常取帧和其他检查 */
ret = OK;

out_unlock:
spin_unlock_irqrestore(&dev->lock, flags);
out_sem:
up(&dev->sem);
return ret;
```

进入 `out_sem` 时，外层信号量被释放了，但 `dev->lock` 没有释放，保存的 IRQ 状态
也没有由配对接口恢复。后续音频中断申请同一把锁时便会等待，所在 CPU 无法正常
退出中断。持锁后的无效主帧、无效参考帧也有类似错误出口。

加锁前检查到“有帧”表示当时的状态，两次检查之间队列可能发生并发变化。持锁后
即使发现队列为空，也必须先解锁并恢复 IRQ 状态，再释放外层信号量并返回。
加锁前就没有帧的分支则不能强行解锁，因为它从未持有这把锁。

IRQ 状态未配对恢复，为什么后来还能进入音频中断？错误出口后仍有内核返回处理；
待处理工作中的 `local_irq_enable()`、返回用户态时的状态恢复等都可能重新允许 IRQ，
驱动持有的自旋锁仍需要通过配对接口释放。

## 4. RCU：为什么对象已经不用了，还不能马上释放

### 4.1 先看 RCU 要解决的并发问题

假设内核有一份共享配置，很多 CPU 都要读取，偶尔有一个写者更新。最直接的做法是
每次读写都先拿同一把锁，保证读者不会访问正在修改或已释放的对象；但在读多写少的
场景中，频繁读也会争抢这把锁。

RCU（Read-Copy-Update）提供另一种思路：读者在受保护的读侧临界区内访问对象；
写者更新共享入口后，**先别释放旧对象**，等可能看见旧对象的读者结束，再回收。
读侧通常无需与写者争抢同一把普通锁，等待主要发生在旧对象的回收阶段。

它解决的是并发访问和对象生命周期问题，并不是“所有操作都不需要锁”。写者之间
仍可能需要锁来保护更新；对象内部若允许并发修改，也必须另有正确的同步规则。
RCU 的名字带有 Copy，但不是每一种使用方式都要复制整个对象；从表中删除旧项并
延迟回收，也可以使用 RCU。

### 4.2 如果立即释放，会发生什么

以下假设共享指针原本指向配置 A，写者准备换成配置 B：

```text
时刻   CPU0：读者                         CPU1：写者
 1     读取共享指针，拿到 A
 2                                        准备 B，把共享指针改为指向 B
 3                                        立即释放 A
 4     按之前拿到的指针读取 A              → 访问已释放内存
```

第 2 步以后，新查找通常会看到 B，但 CPU0 已经拿到了 A 的指针。改变共享指针并不
会自动改变 CPU0 手中保存的旧指针。如果马上释放 A，CPU0 就可能发生 use-after-free
（释放后使用），甚至读到后来复用这块内存的其他数据。

解决思路是把第 3 步推迟：换掉入口后，允许已经看到 A 的读者完成；确认他们不再
依赖 A 后再释放 A。RCU 的宽限期就是用来建立这个回收条件的。

### 4.3 宽限期和静止状态到底是什么

**宽限期**（grace period）不是预先规定的几毫秒。它结束时要提供的保证是：
对于对应的 RCU 类型，在该宽限期开始前已经存在的读侧临界区都已结束。先移除旧
对象的共享入口，再等待覆盖这次移除的宽限期，就可以安全地执行依赖这项保证的回收。

```text
时间 ─────────────────────────────────────────────────────────────→
旧读者：    [进入读侧，取得 A ................. 退出读侧]
写者：                    发布 B → 请求宽限期 .......... 完成 → 释放 A
新读者：                          [进入读侧，读取 B ....]
```

示意中的新读者在换指针后读取 B；它可以与旧读者并存。RCU 不要求系统中所有读者
在同一瞬间全部停止，也不要求把后来开始的所有读者都等完。具体读到哪个版本取决于
并发时序；RCU 的发布和读取接口负责共享指针的访问顺序。

本次设备的配置是 `CONFIG_PREEMPT_NONE`、Tree RCU。对该配置而言，RCU 可利用 CPU
经过符合条件的上下文切换、用户态、空闲等 **静止状态** 来判断旧读侧执行已结束。
一个 CPU 到达适用的静止状态，表示这个 CPU 上相关的旧读侧临界区已经结束；不是说
CPU 断电或物理停止。RCU 收齐本轮需要的安全进展后，才将宽限期标记为完成：

```text
宽限期开始
   ├─ CPU0 经过或被 RCU 识别为已经处于适用的静止状态 → CPU0 条件满足
   └─ CPU1 经过或被 RCU 识别为已经处于适用的静止状态 → CPU1 条件满足
                                                        ↓
                                         本轮条件收齐，宽限期结束
```

两个 CPU 不必同时处于静止状态：CPU1 先满足、后来继续执行，CPU0 随后满足，本轮
仍可结束。已经处于可识别空闲状态的 CPU 可以被识别为满足条件；离线 CPU 的处理
也由 RCU 按其状态判断。可抢占 RCU 还要考虑被抢占的读者，SRCU 有自己的判断方式。

### 4.4 RCU 的核心接口

先看读者怎样保护访问，再看写者怎样发布新对象、回收旧对象。以下说明适用于本文
讨论的普通 RCU；其他 RCU 类型和不同内核配置的约束应查看对应接口文档 [[1]](#ref)。

#### `rcu_read_lock()` 和 `rcu_read_unlock()`

读者用这对接口标记 RCU 读侧临界区的开始和结束。在临界区中取得并使用受 RCU
保护的指针，离开临界区后不能继续使用该指针，除非已通过引用计数等机制取得了
额外的持有权。这不是取得一把与写者共用的普通互斥锁；本次非抢占内核的普通 RCU
读侧临界区也不能执行会睡眠的操作。

```c
rcu_read_lock();
/* 取得并使用受 RCU 保护的指针 */
rcu_read_unlock();
```

#### `rcu_dereference()`

读者在规定的保护条件下通过它读取受 RCU 保护的共享指针。它还处理该读取所需的
顺序约束；取得指针后，仍须在读侧保护或其他有效持有机制下使用对象。

#### `rcu_assign_pointer()`

写者先完整初始化新对象，再通过它发布共享指针，使后续读者能够按 RCU 的访问规则
看到已初始化的数据。它不负责让多个写者互斥；如果有并发写者，还需要写侧锁等
同步方式。

#### `synchronize_rcu()`

调用者等待相应宽限期完成后再继续，适合允许等待的写侧上下文。若写者已经移除旧
入口，等待完成后就可以按相应约定回收旧对象；不能在持有不允许睡眠的自旋锁时
调用它。

#### `call_rcu()`

调用者登记一个回调后即可继续，不必自己等待。满足相应宽限期后，内核再安排回调
执行；因此对象从登记到回调实际执行期间仍要保持有效。回调可能在软中断或进程
上下文执行，不能执行会睡眠的操作。宽限期完成与回调实际执行是两个不同时间点。

### 4.5 使用 RCU 的完整示例

下面的例子中，`config_ptr` 开始时已指向一个分配并初始化好的配置。配置发布后不再
原地修改，多名写者在可睡眠的线程上下文用 `config_lock` 串行更新。

```c
struct demo_config {
    int threshold;
    struct rcu_head rcu;                  /* 异步回收所需的回调记录 */
};

static struct demo_config __rcu *config_ptr;
static DEFINE_MUTEX(config_lock);         /* 写者之间互斥 */

int read_threshold(void)
{
    struct demo_config *cfg;
    int value;

    rcu_read_lock();
    cfg = rcu_dereference(config_ptr);   /* 在读侧保护中取得共享指针 */
    value = cfg->threshold;              /* 假设初始化保证 cfg 非空 */
    rcu_read_unlock();
    return value;                        /* 带走数值，不继续使用 cfg */
}

static void free_config(struct rcu_head *head)
{
    struct demo_config *old = container_of(head, struct demo_config, rcu);
    kfree(old);
}

/* new_cfg 由调用者分配并完整初始化，成功传入后交给此机制管理。 */
void publish_config(struct demo_config *new_cfg)
{
    struct demo_config *old;

    mutex_lock(&config_lock);
    old = rcu_dereference_protected(config_ptr,
                                    lockdep_is_held(&config_lock));
    rcu_assign_pointer(config_ptr, new_cfg);
    mutex_unlock(&config_lock);

    call_rcu(&old->rcu, free_config);     /* 宽限期后调用 free_config */
}
```

#### 读者和写者怎样配合

读者先进入读侧临界区，再用 `rcu_dereference()` 取得指针并访问，退出后只带走
复制出的整数；示例的普通 RCU 读侧没有执行可能睡眠的操作。写者先完整准备新对象，
在写锁保护下发布新指针，再用 `call_rcu()` 登记旧对象的回收。发布和读取接口还
承担必要的内存顺序约束，不能把它们理解为仅仅给普通赋值换了个名字。

假设读者先取得旧配置 A，写者随后发布 B 并调用 `call_rcu()`：写者无需原地等待，
可以继续；A 要等到适用的宽限期完成并且回调实际执行后才释放。读者必须在退出
临界区前结束对 A 的访问；如果需要长期持有指针，还要有额外的引用计数等机制。
示例中写锁解决多个写者的更新冲突，RCU 解决旧对象何时能被安全释放，两者职责不同。

#### 如果写者可以等待

写者也可以在发布新对象、释放写侧锁后，用下面两行代替示例中的 `call_rcu()`：

```c
synchronize_rcu();  /* 等待相应宽限期 */
kfree(old);         /* 此时再释放旧对象 */
```

这两种方式都要求先移除旧入口，再等待安全回收条件。每个旧对象选择一种回收方式；
宽限期的耗时取决于系统进展。

### 4.6 内核怎样判断宽限期完成，回调又在哪里执行

前面的示例解释了接口；再看本次 Tree RCU 的工作过程。它不会为每个对象逐个询问
“读者是否看过你”，而是维护各 CPU 的进展和本轮宽限期需要等待的状态。下面把
双核情况下的过程简化成几个步骤：

```text
1. 写者调用 call_rcu(&old->rcu, callback)
   → 回调被登记，调用者继续；old 还不能释放

2. RCU 需要一个能覆盖该回收请求的宽限期
   → 记录本轮需要哪些 CPU 等参与者提供安全进展

3. CPU0、CPU1 分别经过或被识别为适当的静止状态
   → 相应进展逐级汇总；条件全部满足后，宽限期完成

4. 内核安排执行已经满足条件的 callback
   → 由 callback 释放 old
```

其中 `old->rcu` 是旧对象内嵌的 `struct rcu_head`。RCU 根据 CPU 状态和回调登记
时序安排相应的宽限期，宽限期完成后再安排执行回调。

本版源码中，`rcu_data` 保存每个 CPU 的相关状态和回调队列，`rcu_node` 将进展
分层汇总；`gp_seq` 是宽限期进度序列。进度达到要求后，符合条件的回调还需由 RCU
回调处理路径执行。序列变化、回调已具备执行条件、回调实际执行是有关联但不同的
观察点；如果调试时发现对象未回收，需要分别检查这些环节。

可以在匹配内核的 `kernel/rcu/tree.c` 查看 `__call_rcu()`、
`rcu_report_qs_rdp()`、`rcu_report_qs_rnp()` 和 `rcu_do_batch()`；它们展示的是
具体实现 [[13]](#ref)。第 8.5 节的临时探针记录自身回调的完成情况。

### 4.7 CPU1 为什么不能替卡住的 CPU0 宣布回收安全

RCU 要收集各 CPU 的安全进展。CPU1 可以继续运行并报告自己的进展，CPU0 的进展
则需要由 CPU0 的状态提供。

本次非抢占内核中，CPU0 长时间困在关中断的音频处理路径，RCU 等不到它的安全进展，
相应的宽限期无法完成，文件回收回调随之积压。RCU stall 的原因与诊断方法见文末
参考资料 [[2]](#ref)。

以一份文件对象 A 为例，假设 CPU0 已卡在音频中断中，而 CPU1 还能运行：

| 时刻 | CPU0 | CPU1 和 RCU | 对象 A 的状态 |
| --- | --- | --- | --- |
| 1 | 音频中断等待未释放的自旋锁 | CPU1 的线程关闭指向 A 的 FD | FD 表项被移除，编号可复用 |
| 2 | 仍在中断中等待 | A 的最后一个引用消失，`__fput` 后的 `file_free` 登记 RCU 回调 | A 等待安全回收 |
| 3 | 仍在中断中等待 | RCU 开始一个能覆盖该回收请求的宽限期 | 回调还不能释放 A |
| 4 | 假设本轮仍需要 CPU0 的安全进展，但 CPU0 无法提供 | CPU1 到达适用的静止状态 | CPU1 的条件满足，CPU0 的条件未满足，宽限期未结束 |
| 5 | CPU0 持续卡住 | 后续宽限期无法获得所需进展 | 等待相应宽限期的回调不能执行，A 不能归还 `filp` |

第 2 步中，A 的所有引用最终都已消失，并且最终清理已经运行。`close()` 可能直接释放
最后一个引用，也可能在其他引用随后释放时才到达最终清理；如果其他 FD 或内核
使用者仍持有引用，A 将继续存活。RCU 根据回调登记时序安排覆盖该请求的宽限期。

RCU 跟踪的是各 CPU 的安全进展，避免逐个记录文件对象的读者。即使 CPU0 此时正在
处理音频中断，RCU 仍需要它提供状态进展，才能确认旧读侧临界区已经结束。

如果 CPU0 在卡住前已经满足了某一轮的条件，那一轮仍可能结束；后续需要 CPU0 新
进展的宽限期则会等待。本次受控实验观察到 RCU 进度和探针回调停滞，并伴随 `filp`
增长。其他路径继续分配文件对象时，积压会持续扩大。故障消除后，CPU0 恢复进展，
宽限期和回调随之推进，A 的槽位最终归还缓存。

RCU 背景文章见文末参考资料 [[7]](#ref) [[8]](#ref)。

## 5. FD、struct file、filp：为什么 close 后内存还能增长

### 5.1 从一次 open 开始：编号和对象不是同一样东西

假设程序执行 `fd = open("/proc/meminfo", O_RDONLY)`，系统返回整数 `7`：

```text
应用变量 fd = 7              它只是一个编号
        │
        ▼
进程共享的 FD 表 [7] ─────→ 内核里的 struct file 对象 A
                               ├─ 引用计数 f_count
                               ├─ 访问方式、当前读写位置等状态
                               └─ 指向相关文件操作及其他对象
```

文件描述符（FD）是查表用的整数；`struct file` 是内核表示一次打开状态的对象。
普通文件、设备、socket、pipe 和 proc 文件都可能涉及这种对象。多个 FD 可以通过
`dup()` 等方式引用同一个 `struct file`；同一路径多次独立 `open()` 也可能创建
不同的文件对象。所以 FD 数、路径数和 `struct file` 数不能直接画等号。

引用计数可以理解成“还有多少持有者需要这个对象”。最后一个引用消失前，文件对象
不能进行最终清理；最后一个引用消失后，还需要处理并发查找和延迟回收。

### 5.2 slab 和 filp：对象从哪里分配

内核经常申请大小相同的小对象。slab 分配器为这类对象管理缓存：先组织内存页，再
从中提供对象槽位，回收后也可能暂留空槽位以供复用。`filp` 是本次内核分配
`struct file` 的 slab 缓存名；它不是文件内容缓存，也不是音频帧缓冲。

本次 `/proc/slabinfo` 中每个 `filp` 对象大小是 192 字节，开头字段的意思是：

```text
name  active_objs  num_objs  objsize  objperslab  pagesperslab ...
filp  正在使用数   总槽位数   对象字节数  每个 slab 的对象数  ...
```

例如，一条假设的 `filp 500 1024 192 ...` 表示当前报告 500 个活跃对象、缓存共
1024 个槽位，每个对象 192 字节。500 个活跃对象本身约占 `500 × 192 = 96000`
字节，其余 524 个槽位可供复用。

`active_objs × objsize` 可粗略估算活跃对象本身的字节数；总物理页占用还受空槽位、
对齐和管理开销等影响。对象被释放后，缓存也可能保留空槽位，因此需要结合活跃
对象和页面占用随时间的变化判断。

OOM 日志里的 `filp 18919KB` 是这类缓存的约 18.5 MiB 占用，单位不是对象个数。
`SUnreclaim` 表示普通内存回收不能直接丢弃这类 slab 对象，并不意味着它们在正常
生命周期结束后也无法释放。

### 5.3 close 后，编号和内存分别什么时候释放

本次 Linux 版本的主要路径可以拆成四个阶段：

```text
1. close(7)：从 FD 表移除第 7 项，编号随后可能复用
                   ↓
2. filp_close / fput：减少对象引用；其他引用还在时，不能做最终清理
                   ↓ 最后一个引用消失
3. __fput 清理文件资源 → file_free 减少 nr_files，并提交 RCU 回调
                   ↓ 宽限期完成且回调执行
4. file_free_rcu → kmem_cache_free：对象槽位归还 filp 缓存
```

`__fput` 还可能通过 task_work 或工作队列延后执行。FD 编号可以先复用，文件对象
随后回收；对象槽位归还缓存后，缓存中的空槽位可用于后续分配。

考虑下列假设：每次 `open` 都新分配一个对象，每次 `close` 都到达最终清理，且
文件的 RCU 回调暂时无法执行。FD 7 恰好每次都被复用：

```text
第 1 轮：open → FD 7 指向对象 A；close → FD 7 空闲，A 等待 RCU
第 2 轮：open → FD 7 指向对象 B；close → FD 7 空闲，A、B 等待 RCU
第 3 轮：open → FD 7 指向对象 C；close → FD 7 空闲，A、B、C 等待 RCU
```

每轮最多多用一个 FD 编号，但等待回收的对象增加了。

本版内核的 `file_free()` 在提交 RCU 回调前递减 `nr_files`，因此
`/proc/sys/fs/file-nr` 可能基本稳定，而 `filp` 仍在增长。该计数使用每 CPU 统计；
具体实现见本次匹配的 `fs/file_table.c` [[14]](#ref)。

### 5.4 已有引用计数，为什么还需要 RCU

引用计数能保护 **已经成功取得引用** 的持有者，但取得引用前还有一个窗口：读者
刚从 FD 表拿到指针，尚未成功增加引用；与此同时，另一个 CPU 可能关闭最后一个
引用。如果对象立刻释放，读者连检查引用计数本身都可能访问到已释放内存。

```text
CPU0：查找 FD                         CPU1：关闭 FD
进入 RCU 读侧
从 FD 表取得对象 A 的指针
                                      从表中移除 A，最后一个引用消失
尝试取得非零引用                     安排 A 在宽限期后回收
若成功且表项仍为 A，可安全持有
否则放弃或重试
退出 RCU 读侧
```

RCU 在这段查找窗口中防止对象内存过早回收；引用计数让成功取得的引用可以在退出
读侧临界区后继续持有。匹配内核的 `fs/file.c` 中 `__fget_files()` 还会在取得引用
后复查 FD 表项；如果表项已变化，就放回引用并重试。更新文件表仍有对应的写侧锁。

Linux 5.10 的文件表并发管理见文末参考资料 [[3]](#ref)。

## 6. 把三个机制连回现场

正常情况下，音频线程无论成功还是失败，都应成对释放持有的锁；各 CPU 能继续运行，
RCU 宽限期和回调得以推进，最后一个引用消失的文件对象也能回到 `filp` 缓存。

本次 B051 异常路径则形成以下顺序：

1. **漏解锁**：取帧在持锁后遇到错误，错误出口绕过 `spin_unlock_irqrestore()`。
2. **CPU 失去进展**：后续音频中断等待同一把锁，无法正常返回。AENC 等待数据也可能
   超时，依赖受影响中断的其他功能可能停顿。
3. **RCU 回收受阻**：卡住的 CPU 不能正常提供所需进展，相关宽限期和回调可能停滞。
4. **内存逐渐积压**：其他仍运行的路径继续创建、关闭文件对象，未执行回收的对象占据
   `filp`；FD 数和 `file-nr` 不一定同步增长。
5. **内存耗尽**：如果增长持续且其他回收无法满足分配，系统之后可能触发 OOM。

第 8 节将展开日志中的超时、内核对象采样、跨核栈快照、锁地址与受控实验。

为了理解为什么 OOM 会晚于 CPU 卡住，假设每秒有 100 个文件对象进入待回收队列，
且回收停止，按 192 字节/对象只计算对象本身：

```text
100 × 192 × 60 = 1,152,000 字节/分钟，约 1.10 MiB/分钟
```

实际耗尽时间取决于剩余内存、文件操作频率和其他内存变化。监控程序访问 proc 文件
也会产生文件操作，因此需要控制采样频率。

## 7. 拿到 OOM 日志，应该按什么顺序读

以下摘自用户贴出的 2026-10-26 17:56 故障日志，删除了无关内容：

```text
ot_adec_dec invoked oom-killer: gfp_mask=0x100cca(GFP_HIGHUSER_MOVABLE), order=0
CPU: 1 PID: 12952 Comm: ot_adec_dec ... 5.10.221 #1
...
slab_reclaimable:429 slab_unreclaimable:7145
Normal free:3560kB min:5120kB ... present:65536kB managed:56148kB
...
filp                   18919KB      18919KB
...
Out of memory: Killed process 12930 (eos_sample) total-vm:839728kB,
anon-rss:6216kB, file-rss:0kB, shmem-rss:0kB
```

### 7.1 先分清三个角色

- `invoked oom-killer`：哪个执行线程碰到了这次分配失败。这里是 `ot_adec_dec`。
- `CPU: 1 PID: 12952`：打印这段故障时所在 CPU 和当前线程 ID。
- `Killed process 12930`：内核选中回收内存的进程，属于 EOS 线程组。

触发分配失败的是最后来申请内存的执行者，被杀对象由 OOM 策略选择。
`oom_score_adj` 参与选择被杀进程；`Tainted` 中的 `P`、`O` 分别表示加载过专有
模块和树外模块。

### 7.2 看真实物理占用，别被 total-vm 吓住

进程看到的地址是虚拟地址。它可以为程序、动态库或尚未使用的区域预留很大的地址
范围，但这些地址不一定都对应当前驻留的物理内存。RSS 统计进程当前驻留的内存页；
其中匿名 RSS 主要对应不以普通文件为后备的页面。内核 slab 是系统内核内存，不能
只看某个进程的 RSS 判断它的大小。

内存通常按页管理。本次平台每页 4 KiB；日志中有的字段以页为单位，有的直接以
KiB 为单位，需要先确认字段再计算。以下各项来自同一份 OOM 日志：

| 本例字段 | 正确读法 |
| --- | --- |
| `total-vm:839728kB` | 约 820MiB 虚拟地址空间 |
| `anon-rss:6216kB` | EOS 当时约 6.1MiB 匿名驻留内存 |
| `present:65536kB` | 这个内存区有 64MiB RAM |
| `managed:56148kB` | 其中交给该内存分配器管理的页约 54.8MiB |
| `slab_unreclaimable:7145` | 此处单位是页；按本平台 4KiB/页，约 27.9MiB |
| `filp 18919KB` | 文件对象缓存约 18.5MiB，包含在 slab 占用里 |

这份日志将排查方向指向内核对象积压，可继续检查文件对象的引用计数和回收进度。

完整报告还有 `Tasks state (memory values in pages)` 表，其中 `total_vm`、`rss`
按页计数，`pgtables_bytes` 列则明确是字节。最后 `Killed process` 一行转换成 kB。
内存压力下，被回收的文件页在再次访问时会发生缺页。

### 7.3 看分配条件和调用栈

`order=0` 表示申请 `2^0` 个连续页，在此平台就是一页 4KiB。
`GFP_HIGHUSER_MOVABLE` 是分配约束标志；可用内存还要满足水位、保留页、内存区
和回收条件。本例 `free=3560KiB` 已低于 `min=5120KiB`。

本例栈包含 `do_PrefetchAbort → do_page_fault → filemap_fault → pagecache_get_page`，
说明最后失败发生在取指缺页所需的文件页分配链上。OOM 栈首先展示发生分配失败的
位置。

### 7.4 另一份故障日志

用户另外提供的 `内存不足崩溃.txt` 中，触发者是 CPU0 上的 `nng:aio:expire`，
EOS `anon-rss=23872KiB`，`slab_unreclaimable=2102` 页，约 8.2MiB。
两份日志都出现 AENC 超时，内存统计数值分别记录在各自的日志中。

## 8. 按普通工程方法，一步一步把范围缩小

### 8.1 第一步：固定现场身份，建立时间顺序

先保存原始串口日志，至少包括首次异常之前、卡住期间和完整 OOM 段。
同时记录固件版本、应用 PID、boot ID 和时间。设备校时可能让墙上时间跳变，
用 `/proc/uptime` 或日志中的单调时间辅助排序；重启后的日志另起一轮。

本次接线中 COM17 是 SoC 串口，COM18 是头部板 MCU 串口，取 Linux OOM 现场要看
COM17。串口工具沿用设备已有通信参数，持续保存到主机文件；同一物理串口由一个
采集进程持有。

板端可先读取：

```sh
uname -a
cat /proc/sys/kernel/random/boot_id
cat /proc/uptime
pidof eos_sample
cat /proc/sys/vm/min_free_kbytes
cat /sys/devices/system/cpu/online
```

按时间记录首次 AENC 超时、画面或输入停止、内存开始下降和 OOM 的时点。

在主机上先检查日志编码，再检索。遇到 UTF-16 BOM，先转换副本，保留原文件：

```sh
file serial.log
# 仅在 file 确认是 UTF-16 时执行；否则直接检索原来的文本文件。
iconv -f UTF-16 -t UTF-8 serial.log > serial.utf8.log
rg -n -C 8 'aenc stream select time out|invoked oom-killer|Out of memory|rcu.*stall' serial.utf8.log
```

`rg -C 8` 用于找到位置，随后读取完整上下文和 OOM 报告。

### 8.2 第二步：观察哪个资源在增长，哪个执行环节停了

优先利用已有 EOS `memory detail`、`memory snapshot` 和内核日志，字段含义见
“内存与 OOM 取证”工具说明 [[21]](#ref)。当五分钟摘要覆盖不了故障
变化时，再补低频采样。至少把以下指标放在同一条时间线上：

| 观察对象 | 入口 | 主要回答的问题 |
| --- | --- | --- |
| EOS 匿名内存、RSS、线程数 | `/proc/PID/status` | 应用自身的驻留内存或线程是否持续增长 |
| EOS FD 数、指向 | `/proc/PID/fd/` | 是否有未关闭的描述符累积 |
| 系统内核占用 | `/proc/meminfo` 的 `Slab`、`SUnreclaim` | 内存是否被内核对象吃掉 |
| 文件对象缓存 | `/proc/slabinfo` 的 `filp` | 文件对象数量是否持续增长 |
| 文件计数 | `/proc/sys/fs/file-nr` | 与 FD、filp 是否出现分离 |
| 每核中断计数 | `/proc/interrupts` | 某个核的定时器、音频、UART、DMA 是否停止推进 |
| RCU 软中断计数 | `/proc/softirqs` 的 `RCU` | RCU 处理是否仍有活动 |
| 编码流计数 | `/proc/umap/aenc` | 音频取放是否继续、成功获取与释放是否配对 |

下面的采样脚本每十秒输出一组数据，约三十分钟结束；PID 消失就停止。使用现有
串口、SSH 或 Telnet 把输出保存在主机。

```sh
#!/bin/sh
ai_pid=$(pidof eos_sample)
case "$ai_pid" in
    ''|*' '*) printf '%s\n' '需要确认唯一的 eos_sample PID' >&2; exit 1 ;;
esac
cat /proc/sys/kernel/random/boot_id
cat /proc/sys/vm/min_free_kbytes
cat "/proc/$ai_pid/stat"
ai_sample=0
while [ "$ai_sample" -lt 180 ] && [ -r "/proc/$ai_pid/status" ]; do
    printf '\nsample=%s pid=%s\n' "$ai_sample" "$ai_pid"
    date
    cat /proc/uptime
    grep -E '^(VmRSS|RssAnon|VmSize|Threads):' "/proc/$ai_pid/status"
    if [ ! -r "/proc/$ai_pid/fd" ] || [ ! -x "/proc/$ai_pid/fd" ]; then
        printf 'fd_count=unavailable\n'
    else
        set -- /proc/"$ai_pid"/fd/*
        if [ -e "$1" ] || [ -L "$1" ]; then
            printf 'fd_count=%s\n' "$#"
        else
            printf 'fd_count=0\n'
        fi
    fi
    grep -E '^(MemTotal|MemFree|MemAvailable|Slab|SReclaimable|SUnreclaim):' /proc/meminfo
    awk '$1 == "filp" {print "filp_active=" $2, "filp_total=" $3, "obj_bytes=" $4}' /proc/slabinfo
    printf 'file-nr: '
    cat /proc/sys/fs/file-nr
    cat /proc/interrupts
    grep 'RCU:' /proc/softirqs
    cat /proc/umap/aenc
    ai_sample=$((ai_sample + 1))
    sleep 10
done
```

采样时记录接口读取状态，并结合 PID、boot ID 和 `/proc/PID/stat` 的进程启动时间
核对身份。比较修复前后时，记录内存水位和中断亲和性。

本次观察到 FD 基本稳定，filp 与不可回收 slab 持续增长，CPU0 中断进度停住，
因此继续检查内核回收进展。若 FD 一直增加，可检查 `open/socket/pipe` 与 `close`；
若主要是匿名 RSS 增加，可检查用户态对象、缓存和分配生命周期。

使用独立串口持续采集；必要时在故障前部署一个固定内存、有运行时限的板内观察器，
故障时输出关键快照。

### 8.3 第三步：检查 timeout 所在的真实分支

从日志符号查找源码：

```sh
# 在含 eos、ehal 的 app 目录执行。
rg -n 'get aenc stream select time out|hal_comm_audio_aenc_recv_frame' ehal
rg -n 'hisi_aenc_get_frame' ehal
```

`hal_comm_audio.c` [[17]](#ref) 中，`select()` 返回 0 时打印超时，并在取得编码流
之前返回。调用方 `hal_module_aenc.c` [[18]](#ref) 在错误时释放本次接收缓冲。
继续核查 SDK 获取 FD 和复用 FD 的过程。

沿链路继续向上游看：MPI `ioctl` → AI/AIE 取帧 → 音频中断和队列。
正常调用路径与错误清理路径都要查，重点是“拿到了什么，在哪个出口归还”。

### 8.4 第四步：找真正停住的执行位置

有 shell 时，可以先看目标线程的内核等待位置，`PID`、`TID` 使用本轮实际值：

```sh
cat /proc/PID/task/TID/comm
cat /proc/PID/task/TID/wchan
cat /proc/PID/task/TID/stack
```

`wchan` 和 `stack` 展示线程等待位置和内核栈。有权限且内核支持时，可以触发
SysRq 将任务和 CPU 状态写入内核日志：

```sh
echo w > /proc/sysrq-trigger   # 阻塞任务
echo t > /proc/sysrq-trigger   # 任务状态、相关栈
echo l > /proc/sysrq-trigger   # 各 CPU 的回溯，能力取决于平台实现
```

准备好日志接收后执行；Linux SysRq 说明见文末参考资料 [[9]](#ref)。

GDB/gdbserver 用于查看 EOS 用户线程、对象状态和用户态死锁；内核侧结合栈、符号
和跟踪点分析，必要时使用已配置的 KGDB 或 JTAG。

若已经用匹配的、带符号的 EOS 程序连接到 gdbserver，可以先执行 `info threads`、
`thread apply all bt 12` 获取有界用户栈，保存后 `detach` 恢复进程。
线程停在 `select` 或 `ioctl` 时，继续沿接口检查内核侧的执行状态。

本次自然故障中，显示刷新工作线程停在 `spi_sync` 的等待链上，说明它在等传输完成。
屏幕保持最后一帧、背光仍亮；PID 存活采样显示 OOM 前进程仍在。UART 接收依赖
中断，内核控制台发送可以轮询 FIFO，因此输入失效时内核仍可打印 OOM。

### 8.5 第五步：补上“同一把锁”的证据

本次在错误清理出口增加有限的观测，记录返回码、CPU、IRQ 状态、锁地址和值，再继续
原来的信号量释放。关键记录为：

```text
codex-ai-unbalanced-return pid=1976 cpu=0 result=a0158016
irq_disabled=1 lock=c2159000 word=b683b682

随后：同一锁 word=b684b682，CPU0 心跳停止，CPU1 心跳继续
```

本平台 ticket spinlock 可以理解成排队取号：`next` 是下一个发出的号码，`owner` 是
当前服务的号码。这里低 16 位 `owner=b682`，高 16 位 `next=b683`；两者不等，锁被占用。
随后 `next` 增加到 `b684` 而 `owner` 不变，说明又来了等待者，前面的持有者没有释放。
正常短时间也会看到不相等，**异常是错误返回时还持锁，并且之后长期不前进**。

CPU0 已不能正常响应取栈请求，于是临时观察模块用每核 100ms 心跳，在 CPU1 发现
CPU0 约两秒没更新时复制它的任务原始栈。结合地址、反汇编与正常错误返回回溯，确认：

```text
ai_dma_callback → osal_spin_lock_irqsave → _raw_spin_lock_irqsave
                                   等待锁 c2159000
```

`on_cpu=1` 是运行状态布尔值，CPU 编号由 `cpu=0` 表示。

RCU 另用一个有界探针观察：每个核最多保留一个待完成的测试回调，记录 `gp_seq`、
回调完成数与 pending。`gp_seq` 是内部进度序列，`cb` 和 `pending` 分别是探针的
回调完成数和待完成数。`/proc/codex_rcu_probe` 由本次临时模块创建。

## 9. 没有原厂 C 源码，怎样读懂反汇编

### 9.1 先理解 .o、.ko、ELF 和 .text

`.o` 是编译出来的目标文件，`.ko` 是装入内核的模块文件。原厂提供的 `mod_ai.o`
包含取帧机器码。两类文件使用 ELF 格式，其中 `.text` 存放代码，另有数据、字符串、
符号表及重定位信息。

反汇编把机器码翻译成汇编指令；反编译则尝试恢复类似 C 的结构。本次使用 GNU
Binutils 和控制流分析定位问题。

### 9.2 在编译主机上读取匹配的文件

下面在 `hisilicon/hi3516cv610` 仓库根目录执行，先解包对应 SDK，并让 ARM 工具链
位于 `PATH`。这些命令只读对象文件，把反汇编保存到主机临时目录：

```sh
ai_source_dir="$PWD/sdk/Hi3516CV610_SDK_V1.0.2.0/smp/a7_linux/source"
ai_obj="$ai_source_dir/out/obj/mod_ai.o"
ai_cross=arm-linux-musleabi-
ai_review_dir=$(mktemp -d)

file "$ai_obj"
"${ai_cross}readelf" -h "$ai_obj"
"${ai_cross}readelf" -SW "$ai_obj"
"${ai_cross}readelf" -sW "$ai_obj" > "$ai_review_dir/symbols.txt"
"${ai_cross}objdump" -dr -j .text "$ai_obj" > "$ai_review_dir/mod_ai.dis"
strings -a -t x "$ai_obj" > "$ai_review_dir/strings.txt"

rg -n 'ai_ioctl_get_ai|ai_dma_callback|osal_spin|osal_sem' "$ai_review_dir/mod_ai.dis"
rg -n 'get_frame_inner|busy list|invalid pool_id' "$ai_review_dir/strings.txt"
"${ai_cross}objdump" -dr -j .text --start-address=0x1a74 --stop-address=0x1f84 "$ai_obj"
```

`readelf -h` 应核实 ELF32、ARM、小端及可重定位类型；`-SW` 给出各节在文件中的位置。
符号表中的 `$t` 是 Thumb 代码映射标记，`$a` 表示 ARM 代码，`$d` 表示数据。
`objdump -d` 解码指令，`-r` 同时显示重定位，`-j .text` 限定代码节。
反汇编时需要使用匹配 ARM 架构的工具，并结合代码节中的常量池识别结果。
GNU objdump 参数说明见文末参考资料 [[10]](#ref)。

当前 SDK 工作目录使用 P003。复核历史故障时，把 `ai_obj` 指向诊断归档中的 B051
对象副本；本地临时修复版在 `0x1c32` 为 `e116`，P003 在此处是共用解锁出口的
参数准备指令。当前回归测试直接使用 P003。

### 9.3 从日志和调用者找函数，不只看最近的符号名

两个 ioctl 入口选择普通 AI 或 AIE 队列，调用 `.text+0x1a74` 的公共实现。
内部日志包含 `get_frame_inner`，空队列日志包含 `ai_check_mem_share:879`，
无效帧日志包含 `frame_check_mem_share:846/860`。把调用目标、参数、日志和控制流
结合起来，才能确定这段代码的用途。

此对象裁掉了部分局部符号，工具使用最近的现存符号显示
`ai_ioctl_set_save_file+偏移`。`strings -t x` 输出字符串的文件偏移；结合字符串
所在节、引用和重定位可以找到使用它的代码。

本次 879、846、860 等行号来自驱动内置日志。给原厂定位时提供 SDK 构建版本、
函数或日志名以及指令位置。

### 9.4 沿每个分支画出资源状态

先认识几个本例够用的指令：`mov` 传值，`ldr/str` 读写内存，`cmp` 比较，
`beq/bne/cbz` 按条件跳转，`b` 直接跳转，`bl` 调用函数并保存返回位置。
不必先掌握整套 ARM 指令，先找加锁、解锁和每一个返回出口。

原始对象的关键片段如下，省略无关指令与相邻符号注释：

```text
1bfa:  bl ...                      ; audio_fb_query_busy_num
1bfe:  cbz r0, 1c34               ; 加锁前无帧，去等待/返回路径
1c08:  bl ...                      ; osal_spin_lock_irqsave
       ...                        ; 持锁检查链表
1c1e:  movw r3, #879              ; 空队列日志中的行号
1c2c:  bl ...                      ; OT_LOG
1c30:  ldr r5, [...]              ; 准备错误码
1c32:  e7a0  b.n 1b76            ; 出错却直接去释放信号量

1b76:  mov r0, r8
1b78:  bl ...                      ; osal_sem_up

1e62:  mov r0, r9                 ; 已有的公共解锁出口
1e64:  add r1, sp, #32            ; 保存的 IRQ flags 地址
1e66:  bl ...                      ; osal_spin_unlock_irqrestore
1e6a:  b.n 1b76                   ; 解锁后再释放信号量
```

在未链接的目标文件里，外部调用可能显示 `bl 0`；下一行 `R_ARM_THM_CALL` 才写明
将由链接器或加载器填入的真实目标，例如 `osal_sem_up`。

读到这里，缺陷可以用纸笔验证：从 `1c08` 成功加锁开始，沿空队列分支走到 `1c32`，
再到 `1b76`，途中没有调用解锁。无效帧共用出口 `1da8` 也有相同问题。
相反，加锁前的无帧路径根本没持有锁，不能把所有错误都强行送进解锁函数。

## 10. 为什么修改 Thumb 指令就能修复

### 10.1 CPU 执行的是机器码

普通 C 源码改动，最终也会经编译变成机器码。本次对应函数只有机器码交付，
因此在已确认的版本上直接改变错误分支的目标，实现“出错先解锁，再释放信号量”。

Thumb 是 ARM 的指令编码方式；本平台使用的 Thumb-2 可以混合 16 位与 32 位指令。
本平台 CPU 使用 32 位 ARM 架构。
本次三处改动恰好都使用 16 位指令，每处 2 字节，共 6 字节。
Arm 对 Thumb 指令宽度的说明见文末参考资料 [[11]](#ref)。

### 10.2 第一处跳转可以手算核对

先把补丁改变的路径画出来。这里的地址都是 B051 对象中 `.text` 节内的位置：

```text
原路径： 1c32: b.n 1b76 ─────────────→ 释放信号量 → 返回
                                    （绕过自旋锁解锁）

修复后： 1c32: b.n 1e62 → 恢复参数 → 释放自旋锁并恢复 IRQ 状态
                              → 1b76 释放信号量 → 返回
```

`b.n` 是一条 16 位的无条件跳转指令；指令中的位字段保存的是相对位移，而非完整
目标地址。修改指令编码后，CPU 在同一处执行时会跳到不同的位置。反汇编中显示的
`e7a0`、`e116` 是按 16 位数值表示的机器码；小端文件中的两个字节顺序在后面说明。

本例使用 16 位无条件 `b.n`，目标与当前指令的位置关系是：

```text
目标 = 当前指令地址 + 4 + 符号扩展后的 11 位立即数 × 2

当前地址 = 0x1c32
新目标   = 0x1e62
立即数   = (0x1e62 - (0x1c32 + 4)) / 2 = 0x116
指令编码 = 0xe000 | 0x116 = 0xe116
```

原指令也可以反算：`0xe7a0` 的 11 位立即数为 `0x7a0`；按 11 位有符号数解释是
`-96`，乘 2 得到 `-192` 字节，因此 `0x1c32 + 4 - 192 = 0x1b76`。新立即数
`0x116` 为正数，乘 2 后向更高地址跳转 556 字节，到达 `0x1e62`。

这条指令的位移范围为相对基址 `-2048` 至 `+2046` 字节，且目标要满足半字对齐。
本例满足条件。
`0xe116` 在小端文件中存成字节 `16 e1`；原来的 `0xe7a0` 则是 `a0 e7`，跳回 `0x1b76`。
通过汇编器和反汇编器解码并验证跳转。GNU CGEN 的 Thumb 分支编码定义见文末
参考资料 [[12]](#ref)。

### 10.3 为什么是三处，而不只改一处

| `.text` 内位置 | 修改前 | 修改后 | 要达到的效果 |
| --- | --- | --- | --- |
| `0x1c32` | `e7a0`，跳 `1b76` | `e116`，跳 `1e62` | 空队列错误先解锁 |
| `0x1da8` | `e6e5`，跳 `1b76` | `e05b`，跳 `1e62` | 无效主帧、参考帧错误先解锁 |
| `0x1dd4` | `4648`，`mov r0,r9`，随后局部解锁 | `e72c`，跳 `1c30` | NULL 路径改走一次公共解锁，避免重复解锁 |

第三处很关键：原来的 `get_busy(NULL)` 路径先在 `1dd4` 附近解锁，再跳到 `1c30`。
如果只把 `1c32` 改成去解锁，它就会解锁两次。第三处使它跳过原来的局部解锁，
准备错误码后只执行公共出口的一次解锁。

复用 `1e62` 还需要检查：此时 `r9` 仍是正确的锁封装地址，栈上保存的 IRQ flags
仍有效，错误码不会被破坏。
本次不新增或删除指令长度，不移动后面的代码，不改函数调用接口和原有错误码。

### 10.4 指令偏移不等于文件偏移，更不等于运行地址

本例 `mod_ai.o` 的 `.text` 在文件偏移 `0x34`，所以该对象中第一处修改在文件内是
`0x34 + 0x1c32 = 0x1c66`。`ot_ai.ko` 的节布局和加载后的虚拟地址需要分别计算。

当时的临时脚本 `fix_ai_get_frame_lock.py` 解析 ELF 节表，校验整个 `.text` 的摘要
和原指令，只接受已核对的 B051 代码，修改后再次核对摘要；已修复则直接识别，
未知版本报错。构建同时处理 `mod_ai.o` 和 `ot_ai.ko`，避免下一次链接从旧对象
重新生成有缺陷的模块。接入原厂 P003 后，该脚本及其构建调用已删除。

这是一种针对确定二进制版本的修复方式。拥有源码时，应在 C 的错误出口补齐清理并
重新构建。海思 P003 已通过独立验证，当前 overlay 保存它的原始对象和模块，
不再修改指令。上面的三个偏移对应历史 B051 版本。

## 11. 修复验证

### 11.1 分别验证控制流与真实硬件

先运行真实厂商 ARM object，把依赖的锁、队列等替换成可计数的测试实现。
这样能确定某条分支的加锁/解锁次数和返回码，不需要靠偶发时序才能命中。
已有入口是回归测试 [[19]](#ref) 及其 fixture [[20]](#ref)：
当前测试直接使用 overlay 中的 P003 对象，覆盖 AI、AIE、RAW 三个入口、六种异常
路径及两种调用前 IRQ 状态，共 36 组；历史 B051 对照结果保留在诊断证据中。

```sh
# 在固件仓库根目录执行；需要 ARM musl 工具链与 qemu-arm[-static]。
CROSS_COMPILE=arm-linux-musleabi- python3 -m unittest discover -s tests -p test_ai_get_frame_lock.py -v
```

QEMU 执行厂商 ARM 指令，锁、IRQ 和队列由 fixture 模拟。测试结果应显示用例已执行。

| 异常路径 | 原始对象的结果 | 修复应满足的条件 |
| --- | --- | --- |
| 加锁前就无帧 | 未加锁，也未解锁 | 保持正确，不能多解锁 |
| 加锁后队列为空 | 加锁 1 次、解锁 0 次 | 加锁与解锁各 1 次 |
| 主帧无效 | 加锁 1 次、解锁 0 次 | 加锁与解锁各 1 次 |
| 参考帧无效 | 加锁 1 次、解锁 0 次 | 加锁与解锁各 1 次 |
| 拷贝失败 | 原本已解锁 | 保持只解锁 1 次 |
| 取 busy 节点返回 NULL | 原本已解锁 | 保持只解锁 1 次 |

板端通过暂停和恢复应用制造调度压力，抓到了 AIE 帧队列为空、取帧错误返回时
自旋锁仍被持有的状态。此后串口和网络失去响应。

板端受控对照在原有“查计数到加锁”的窗口增加 1ms 延迟，配合调度压力，提高异常
分支命中率，并观察应用恢复后的锁状态、中断和取帧进展。

### 11.2 修复前后的观察结果

| 观察结果 | 对应现象 |
| --- | --- |
| OOM 中 filp 很大 | 文件对象占了大量内核内存 |
| FD 稳定、filp 增长 | 描述符数与对象释放出现分离 |
| RCU 序列、探针回调停住 | 回收进展受阻 |
| 错误返回仍持锁，IRQ 随后等待同一把锁 | 驱动清理缺陷导致中断处理卡住 |
| 修复后命中相同错误分支仍正常返回 | 补齐解锁后，中断和取帧继续运行 |

本次修复版在相同受控条件下运行十分钟、1602 轮压力，至少命中 153 次原空队列
分支，两核、RCU 和文件回收继续推进。随后不含诊断代码的修复模块连续采样约
三十分钟，61 组样本中 FD 固定为 54，filp 在 438–945 之间变化，没有持续积压。
这些数字说明正常的 filp 波动不必降到零，关键是回收能够继续。

上述数据来自 2026-10-27 的临时修复验证。长时间观察时辅助库仍有映射，
`min_free_kbytes` 临时设为 5120。清理并重启后的默认环境记录约一分钟，观察了启动
和基本功能；海思 P003 随后进行了单独的板端验证。

## 12. 排查方法

排查时可以按以下顺序使用日志和工具：

1. 用串口保存完整现场，按版本、boot ID、PID 分轮，找出最早失去进展的位置。
2. 用 OOM 的内存分布确定方向，再用连续采样区分匿名内存、FD 和内核对象的增长。
3. 查错误日志对应分支，沿调用者、资源申请与释放路径阅读代码，逐个排除假设。
4. 在中断和 RCU 都失去进展时转向内核，获取栈与同一资源的地址，不停留在用户态超时。
5. 用 `readelf`、`objdump`、`strings` 和源码/厂商手册还原分支，在纸上标注每条路径
   的持锁状态。只有二进制时，向厂商提交精确构建信息、日志行号和错误跳转位置。
6. 设计能命中问题分支的最小实验，做原始版与修复版对照，再回到正常产品环境观察。

有源码且调试配置允许时，可以借助 lockdep、`CONFIG_DEBUG_SPINLOCK`、ftrace 或
跟踪点分析锁依赖和执行状态。二进制驱动可向厂商申请匹配符号或修复。

这次排查围绕三个问题逐步验证：
**内存到底积压在哪里；哪个执行环节阻止了释放；修改那个环节后，同样的错误分支能否正常退出。**

## ref

### 本次故障原始记录

本机原始证据目录为 `~/.cache/eos/diagnostics/20261027-freeze/`。

| 需要复核的事实 | 原始记录或代码 |
| --- | --- |
| 自然卡死至 OOM | `fault-1248-full-serial.log` |
| AIO 中断的 CPU 分布与 IRQ 名称 | `irq-scheduling.txt`、`audio-kernel-identity.txt` |
| AI 通道 normal 模式与取帧线程 | `schedule-stress-preflight.txt`、`cpu0-gdb-small.txt` |
| 调度压力下的空队列与漏解锁 | `fault-1552-raw.log`、`investigation.txt` |
| 漏解锁、同锁等待、心跳和 filp 变化 | `longwindow-serial-081720-raw.log` |
| 音频中断分发和取帧处理的反汇编 | `ot-aio-disassembly.txt`、`ot-ai-disassembly.txt` |
| 临时错误出口观察和 1ms 窗口 | `ai-trace-window/ai_trace.c`、`ai_trace_entry.S` |
| 跨核原始栈快照 | `cpu-watch/codex_cpu_watch.c` |
| 有界 RCU 回调探针 | `probe/codex_rcu_probe.c` |
| 修复前后 ARM 异常路径 | `ai-lock-proof-six-cases.txt`、`ai-lock-proof-fixed-result.txt`，以及前述回归测试 |
| 十分钟压力与长时间采样结果 | `fixed-pressure-result.json`、`release-acceptance-result.json` |

匹配内核源码位于相邻固件仓库已解包 SDK 的
`open_source/linux/linux-5.10.y/`。检查文件对象回收时看 `fs/file_table.c`，文件查找与
关闭看 `fs/file.c`，自旋锁实现看 `arch/arm/include/asm/spinlock.h`，串口输入与
控制台输出差异看 `drivers/tty/serial/amba-pl011.c`。

### 外部参考资料

1. Linux 内核文档 What is RCU?：[Linux 5.10 版本](https://www.kernel.org/doc/html/v5.10/RCU/whatisRCU.html)、[当前版本](https://docs.kernel.org/RCU/whatisRCU.html)。
2. [Linux 5.10：Using RCU’s CPU Stall Detector](https://www.kernel.org/doc/html/v5.10/RCU/stallwarn.html)，RCU stall 的可能原因。
3. [Linux 5.10：File management in the Linux kernel](https://www.kernel.org/doc/html/v5.10/filesystems/files.html)，文件表、引用计数与 RCU。
4. [Linux 5.10：Locking lessons](https://www.kernel.org/doc/html/v5.10/locking/spinlocks.html)，自旋锁及 IRQ 保存、恢复。
5. [Linux 5.10：Mutex Design](https://www.kernel.org/doc/html/v5.10/locking/mutex-design.html)，mutex 使用条件。
6. [Linux 5.10：Linux generic IRQ handling](https://www.kernel.org/doc/html/v5.10/core-api/genericirq.html)，硬件中断及驱动处理接口。
7. [Linux Kernel Internals：RCU](https://kernel-internals.org/locking/rcu/)，RCU 概念、接口与示例。
8. [深入理解 Linux 内核 RCU 机制与 eBPF 无锁内存分配](https://blog.csdn.net/mounter625/article/details/163383096)，RCU 和 eBPF 内存分配。
9. [Linux 5.10：Linux Magic System Request Key Hacks](https://www.kernel.org/doc/html/v5.10/admin-guide/sysrq.html)，SysRq 操作说明。
10. [GNU objdump 参数说明](https://sourceware.org/binutils/docs/binutils/objdump.html)，反汇编和重定位选项。
11. [Arm：Instruction width specifiers](https://support.arm.com/documentation/dui0489/i/arm-and-thumb-instructions/instruction-width-specifiers)，ARM 和 Thumb 指令宽度。
12. [GNU CGEN：Thumb 分支编码](https://sourceware.org/cgen/gen-doc/arm-thumb-insn.html#insn-b)，本例使用的 16 位跳转格式。

### 匹配源码和工具

以下链接指向相邻仓库和解包的 SDK 源码。

13. [Linux 5.10 `kernel/rcu/tree.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/kernel/rcu/tree.c)：宽限期和回调处理。
14. [Linux 5.10 `fs/file_table.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/fs/file_table.c)：文件对象清理和回收。
15. [Linux 5.10 `fs/file.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/fs/file.c)：FD 查找、引用和关闭。
16. [Hi3516CV610 设备树 `hi3516cv610.dtsi`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610.dtsi)：`aiao` 设备及 AIO 中断。
17. [EHAL `hal_comm_audio.c`](../../../../app/ehal/src/backends/hisi/comm/hi3516cv610/hal_comm_audio.c)：AENC 等待和超时日志。
18. [EHAL `hal_module_aenc.c`](../../../../app/ehal/src/backends/hisi/modules/hal_module_aenc.c)：AENC 接收路径与资源处理。
19. [AI 取帧锁回归测试](../../../../hisilicon/hi3516cv610/tests/test_ai_get_frame_lock.py)。
20. [AI 取帧锁测试 fixture](../../../../hisilicon/hi3516cv610/tests/fixtures/ai_get_frame_lock.c)。
21. [EOS 内存与 OOM 取证工具说明](../../../../app/eos/tools/README.md#内存与-oom-取证)。
22. [Linux 5.10 ARM 自旋锁实现](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/arch/arm/include/asm/spinlock.h)。
23. [Linux 5.10 `amba-pl011.c`](../../../../hisilicon/hi3516cv610/sdk/Hi3516CV610_SDK_V1.0.2.0/open_source/linux/linux-5.10.y/drivers/tty/serial/amba-pl011.c)：串口驱动与控制台。

### normal 模式与取帧调度

24. [海思 Hi3516CV610 音频开发参考](https://docs.hisilicon.com/projects/Hi3516CV610/main/api-reference/MPP/MPP%20%E5%AA%92%E4%BD%93%E5%A4%84%E7%90%86%E8%BD%AF%E4%BB%B6%20V6.0%20%E5%BC%80%E5%8F%91%E5%8F%82%E8%80%83/7%20%E9%9F%B3%E9%A2%91.html)：AI 通道工作模式、VQE 和取帧接口。
25. [海思 Hi3516CV610 性能调优指南](https://docs.hisilicon.com/projects/Hi3516CV610/main/samples/%E8%B0%83%E8%AF%95%E8%B0%83%E4%BC%98/MPP/%E6%80%A7%E8%83%BD%E2%95%B1%E5%B8%A6%E5%AE%BD%E2%95%B1%E5%BB%B6%E6%97%B6%E2%95%B1%E5%86%85%E5%AD%98%E8%B0%83%E4%BC%98%E6%8C%87%E5%8D%97/%E6%80%A7%E8%83%BD%E2%95%B1%E5%B8%A6%E5%AE%BD%E2%95%B1%E5%BB%B6%E6%97%B6%E2%95%B1%E5%86%85%E5%AD%98%E8%B0%83%E4%BC%98%E6%8C%87%E5%8D%97.html)：AI 工作模式与 `ot_ai_get` 线程。
26. [EHAL `hal_core_pipeline_resources.c`](../../../../app/ehal/src/core/hal_core_pipeline_resources.c)：音频桥接线程的取帧与送编码流程。
27. [EHAL `hal_comm_audio_policy.c`](../../../../app/ehal/src/backends/hisi/comm/hi3516cv610/hal_comm_audio_policy.c)：取帧错误码转换。
28. 海思 FAE 对 normal 模式下高负载影响取帧的反馈。
