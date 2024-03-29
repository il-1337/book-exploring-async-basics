# Epoll, Kqueue and IOCP

有一些相当知名的库使用Epoll、Kqueue和IOCP实现了一个跨平台的事件队列，用于在Linux、Windows和Mac上进行操作。

Nodes运行时的一部分是[libuv](https://github.com/libuv/libuv)，它是一个跨平台的异步I/O库。`libuv`不仅在Node中使用，还构成了[Julia](https://julialang.org/)和[Pyuv](https://github.com/saghul/pyuv)创建跨平台事件队列的基础。大多数编程语言都有它的绑定。

在Rust中，我们有[mio - Metal IO](https://github.com/tokio-rs/mio)。`Mio`驱动了[Tokio](https://github.com/tokio-rs/tokio)中使用的操作系统事件队列，Tokio是一个提供I/O、网络、调度等功能的运行时。`Mio`对于`Tokio`的作用类似于`libuv`对于`Node`的作用。

`Tokio`支持许多Web框架，其中包括[Actix Web](https://github.com/actix/actix-web)，这是一个性能非常好的框架。

由于我们想要了解一切是如何工作的，我不得不创建一个极其简化的事件队列的版本。我将其称为`minimio`，因为它受到了`mio`的极大启发。

> 我将写一本简短的书（比这本书要短得多），详细介绍这个工作原理，如果你感兴趣的话，你可以访问它的[GitHub存储库中](https://github.com/cfsamson/examples-minimio)的代码。这本书还将简要介绍`wepoll`，它被用作`mio`和`libuv`中IOCP的优化替代方案。

然而，在这里我们将为每个项目进行简要介绍，以便你了解基础知识。

## 为什么使用基于操作系统的事件队列呢？

如果你记得我之前的章节，你就会知道我们需要与操作系统密切合作，以使 I/O 操作尽可能高效。像 Linux、MacOS 和 Windows 这样的操作系统提供了多种执行 I/O 的方式，包括阻塞和非阻塞。

因此，阻塞操作对于我们作为程序员来说是最不灵活的，因为我们将控制权交给操作系统，操作系统会挂起我们的线程。其最大优势在于，一旦我们等待的事件准备就绪，我们的线程就会被唤醒。

非阻塞方法更加灵活，但需要一种告诉我们任务是否准备就绪的方法。这通常通过返回某种数据来实现，表示任务是**就绪**还是**未就绪**。其中一个缺点是我们需要定期检查这种状态，以便判断状态是否发生了变化。

Epoll、kqueue 和 IOCP 是一种结合了非阻塞方法灵活性而又没有其缺点的方法。

> 我们不会涵盖诸如`poll`和`select`这样的方法，但是我有[一篇文章](https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)供你参考，如果你想了解一下这些方法以及它们与`epoll`的区别。

## 基于准备就绪的事件队列

Epoll 和 Kqueue 是我们所谓的准备就绪的事件队列。它们之所以被称为准备就绪，是因为它们会告诉你何时可以执行某个动作。例如，当套接字准备好可以进行读取时。

**基本上，当我们使用 epoll/kqueue 从套接字读取数据时，会发生以下步骤：**

1. 通过调用系统调用`epoll_create`或`kqueue`来创建一个事件队列。
2. 我们请求操作系统提供表示网络套接字的文件描述符。
3. 我们使用第二个系统调用，在我们创建的事件队列（步骤1）中注册对该套接字上“读取”事件的兴趣。
4. 接下来，我们调用`epoll_wait`或`kevent`来等待事件 - 这将阻塞（暂停）调用它的线程。
5. 当事件准备就绪时，我们的线程将被解除阻塞（恢复），并且我们从“等待”调用中返回有关发生的事件的数据。
6. 我们在步骤2中创建的套接字上调用`read`。

## 基于完成的事件队列

IOCP（I/O 完成端口）是一种事件队列，它提供了在事件完成时收到通知的功能。例如，数据已经被读取到了一个缓冲区。

以下是事件队列的基本操作步骤：

1. 通过调用系统调用`CreateIoCompletionPort`来创建一个事件队列。
2. 创建一个缓冲区，并请求操作系统给我们一个套接字的句柄。
3. 使用另一个系统调用向该套接字注册对“读取”事件的关注，同时将我们在步骤（2）中创建的缓冲区传递给操作系统，用于存储读取到的数据。
4. 接下来，调用`GetQueuedCompletionStatusEx`方法，该方法将阻塞当前线程，直到事件完成。
5. 线程被解除阻塞，并且我们的缓冲区被填充了我们感兴趣的数据。


## Epoll 

`Epoll`是`Linux`实现事件队列的方式。在功能上，它与`Kqueue`有很多相似之处。使用`epoll`而不是Linux上的其他类似方法（如`select`或`poll`）的优势在于，`epoll`被设计为能够非常高效地处理大量事件。

### Kqueue

`Kqueue`是macOS实现事件队列的方式，实际上，它是BSD使用的方式，而macOS也在使用。就高级功能而言，它在概念上与 `epoll`相似，但使用方式有所不同。

有人认为它使用起来略微复杂，抽象性和“通用性”更强。

### IOCP

`IOCP`或输入输出完成端口是Windows处理此类事件队列的方式。

**完成端口**将通知您事件已经**完成**。现在，这可能听起来像是一个小的区别，但实际上并非如此。当您想要编写一个库时，这一点尤其明显，因为在两者之间进行抽象意味着您要么将`IOCP`建模为**准备就绪型**，要么将`epoll/kqueue`建模为完成型。

向操作系统提供缓冲区也会带来一些挑战，因为在等待操作返回时，保持此缓冲区不被触及非常重要。

> My experience investigating this suggests that getting the `readiness based`
> models to behave like the `completion based` model is easier than the other
> way around. This means get IOCP to work first and then fit `epoll` and `kqueue`
> into that design.

> 我调查此问题的经验表明，使**准备就绪型**模型行为类似于**完成型**模型比反之要容易。这意味着首先让`IOCP`工作，然后再将`epoll`和`kqueue`纳入该设计中。