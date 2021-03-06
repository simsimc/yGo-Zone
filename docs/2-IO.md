“一切皆文件”，Linux 内核会把所有外部设备都看作一个文件来操作。在网络 I/O 中系统对一个 Socket 的读写也会有相应的描述符，称为 socket fd（Socket 描述符）。

如下图以 Socket 读取数据 recvfrom 调用为例，它整个 I/O 流程分为两个阶段（*非常重要*）：
1. 等待 Socket 数据准备好；
2. 将数据从内核拷贝到应用进程中。

![recvfrom](https://static001.geekbang.org/resource/image/f8/20/f872176205d997f30753ab87a276fa20.png)


## 同步、异步、阻塞、非阻塞
### 同步与异步
首先解释同步和异步的概念，这两个概念与消息的**通知机制**有关。也就是说同步和异步主要是从消息通知机制角度来说的。
#### 同步
同步就是一个任务的完成需要依赖另一个任务时，只有等待被依赖的任务完成后，以来的任务才能算完成，这就是一种可靠的任务序列。（两个任务状态相同，成功/失败状态保持一致）
#### 异步
异步是不需要等待被依赖的任务完成的，只要通知被依赖的任务要完成什么工作，依赖的任务也立即执行，只要自己的任务完成就算完成，所以他是不可靠的任务序列。（两个任务状态可能不同）
#### 区别
当一个同步调用发出后，调用者要一直等待返回消息通知后，才能进行后续的执行；而一个异步过程调用发起后，调用者不会立刻得到返回结果，而是执行部件通过状态、通知或者回调来通知调用者任务完成，在这个调用者被通知前，可以执行其他消息。

这里提到执行部件和调用者通过三种途径返回结果：状态、通知和回调。使用哪一种通知机制依赖于执行部件的内部实现。

- 如果执行部件使用状态来通知，那么调用者就需要每隔一定时间检查一次，效率很低；
- 如果使用通知或者回调函数，执行部件无需做额外的操作，效率很高；

### 阻塞与非阻塞
阻塞和非阻塞这两个概念与程序（线程）**等待消息通知时的状态**（无所谓同步或异步）有关。也就是说阻塞与非阻塞主要是程序（线程）等待消息通知时的状态角度来说的。

阻塞的核心就是**线程的挂起**。

#### 阻塞
阻塞调用是指调用结果返回之前，当前线程会被挂起，一直处于等待消息通知，不能够执行其他业务。

#### 非阻塞
非阻塞指在不能立即得到结果之前，该函数不会阻塞当前线程，而会立刻返回。

#### 区别
虽然看起来非阻塞可以提高CPU的利用率，但是也带来了另一个后果就是系统的线程切换增加。增加的CPU执行时间是否能不畅系统切换的成本需要仔细评估。

### 两两组合
#### 同步阻塞
效率是最低的，专心做一件事，其他的什么都不做。**在实际程序中：** 就是未对fd设置`O_NONBLOCK`标志位的read/write操作；
#### 异步阻塞
异步操作是可以被阻塞的，只不过他不是在处理消息时被阻塞，而是在等待消息时被阻塞。
可以参考`select`函数，如果timeout参数为NULL，那么如果所关注的事件没有一个被触发，那程序就一直会阻塞在`select`函数上。

#### 同步非阻塞
效率低下，一边做当前的事情，一边需要关注是否有返回结果，程序需要在两种不同的行为之间来回切换。
#### 异步非阻塞
效率更高，做当前事情的同时，无需定时关注是否有返回结果，有返回结果时会有对应的消息机制告知。

### 区别
#### 阻塞和同步的区别
既然同步是等待消息返回，阻塞也是类似，那他们的区别是什么呢？
因为很多时候同步操作会以阻塞的形式表现出来。比如阻塞的read/write操作中，其实是把消息通知机制和等待消息通知的状态结合在一起了，在这里，所关注的消息就是fd是否可读/写，而等待消息通知的状态则是fd可读/写等待过程中程序的状态。当我们将这个fd设置为非阻塞的时候，read/write操作就不会在等待消息通知这里阻塞，如果fd不可读/写则立即返回。
同理，也会有人把异步和非阻塞混淆，因为异步操作一般不会再真正的IO操作处被阻塞，比如用`select`函数，当`select`返回可读时再去执行read一般都不会被阻塞，而是在`select`函数调用处阻塞。

对于同步调用来说，很多时候当前线程可能还是激活的，只是从逻辑上，当前函数没有返回而已，此时，这个线程是可以去处理其他消息的，但阻塞不行。若这个线程在等待当前函数返回时：
1. 仍然在执行其他消息处理，那么这种情况叫做，同步非阻塞；
2. 没有执行其他消息处理而是处于挂起状态，那么叫做，同步阻塞；

所以同步的实现方式会有两种：同步阻塞、同步非阻塞；同理异步也会有两种实现：异步阻塞、异步非阻塞。

### 核心点
1. **理解“消息通知机制”和“等待消息通知时的状态”这两个概念，这是理解四个概念的关键所在。**
2. **异步操作是可以被阻塞的，只不是他不是在处理消息时阻塞，而是在等待消息通知时被阻塞。** 

## Linux 硬中断和软中断
### 硬中断

由与系统相连的外设(比如网卡、硬盘)自动产生的。主要是用来通知操作系统系统外设状态的变化。比如当网卡收到数据包的时候，就会发出一个中断。我们通常所说的中断指的是硬中断(hardirq)。

### 软中断

为了满足实时系统的要求，中断处理应该是越快越好。linux为了实现这个特点，当中断发生的时候，硬中断处理那些短时间就可以完成的工作，而将那些处理事件比较长的工作，放到中断之后来完成，也就是软中断(softirq)来完成。

### 中断嵌套

Linux下硬中断是可以嵌套的，但是没有优先级的概念，也就是说任何一个新的中断都可以打断正在执行的中断，但同种中断除外。软中断不能嵌套，但相同类型的软中断可以在不同CPU上并行执行。

### 相关文章

- [同步、异步、阻塞、非阻塞](https://zhuanlan.zhihu.com/p/371508875)
- [聊聊Linux 五种IO模型](https://www.jianshu.com/p/486b0965c296)
- [Linux硬中断和软中断](https://zhuanlan.zhihu.com/p/85597791)