# 什么是Node？

我们必须先简要解释一下什么是Node，以确保我们在同一页面上。

Node 是一个 JavaScript 运行时，允许 JavaScript 在您的桌面（或服务器）上运行。JavaScript 最初设计为浏览器的脚本语言，这意味着它依赖于浏览器来解释它并为其提供运行时。

这也意味着桌面上的 JavaScript 需要被解释（或编译），并且需要提供运行时才能执行任何有意义的操作。在桌面上，[V8 JavaScript 引擎](https://en.wikipedia.org/wiki/V8_JavaScript_engine)编译 JavaScript，而[Node](https://en.wikipedia.org/wiki/Node.js)提供运行时。

从语言设计的角度来看，JavaScript 有一个优势：一切都设计为异步处理。正如您现在已经了解的那样，如果我们想要充分利用硬件，特别是如果您有大量的 I/O 操作要处理，这是非常关键的。

一个典型的场景是 Web 服务器。Web 服务器处理大量的 I/O 任务，无论是从文件系统读取还是通过网络卡通信。

## 为什么选择 Node

- 在浏览器进行Web开发时，无法避免使用JavaScript。在服务器端使用JavaScript允许程序员在前端和后端开发中使用相同的语言
- 后端和前端之间存在代码重用的潜力
- Node 的设计使其能够创建非常高性能的Web服务器
- 当您只使用JavaScript处理时，使用Json和Web API非常容易

## 有用的事实

让我们首先驳斥一些谬论，这可能会使我们在开始编写代码时更容易理解。

### JavaScript事件循环

JavaScript是一种脚本语言，单独无法做太多事情。它没有一个事件循环。在Web浏览器中，浏览器提供了一个运行时，其中包括一个事件循环。而在服务器端，Node提供了这种功能。

你可能会说，JavaScript作为一种语言，如果没有某种形式的事件循环，将很难运行（由于它的基于回调的模型），但这并不重要。

### Node是多线程的

与我在几次见到的声明相反，Node使用线程池，因此它是多线程的。然而，Node中“推进”您的代码的部分确实在单个线程上运行。当我们说“不要阻塞事件循环”时，我们指的是这个线程，因为这将阻止Node推进其他任务。

我们将看到为什么阻塞这个线程是一个问题，以及如何处理它。

### V8 JavaScript引擎

现在，这是我们需要重点关注的地方。V8引擎是一个JavaScript即时编译器（JIT编译器）。这意味着当您编写一个`for`循环时，V8引擎会将其转换为在您的CPU上运行的指令。有许多JavaScript引擎，但最初Node是在V8引擎之上实现的。

V8引擎本身对我们来说并没有什么用，它只是解释我们的JavaScript。它无法执行I/O、设置运行时或类似的操作。只用V8编写JavaScript会是一种非常有限的体验。

> 因为我们写的是Rust（尽管我们让它看起来有点像JavaScript），所以我们不会涵盖解释JavaScript的部分。我们只专注于Node如何工作和处理并发，因为这是我们目前的主要关注点。

### Node事件循环

Node内部将其真正的工作分为两类：

#### I/O密集型任务

主要等待某些外部事件发生的任务由在`libuv`中实现的跨平台epoll/kqueue/IOCP事件队列处理，在我们的例子中由`minimio`实现。

#### CPU 密集型任务

主要由CPU密集型的任务处理的线程池。该线程池的默认大小为4个线程，但可以通过Node运行时进行配置。

无法由跨平台事件队列处理的I/O任务也在此处处理，这是我们示例中使用的文件读取的情况。

大多数Node的C++扩展使用此线程池执行其工作，这是它们用于计算密集型任务的许多原因之一。

## Further information

If you do want to know more about the Node eventloop I have one short page of the `libuv` documentation I can
refer you to and two talks for you that I find great (and correct) on this subject:

[Libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html#design-overview)

This first talk one is made held by [@piscisaureus](https://github.com/piscisaureus) and is an excellent 15 minute overview - i especially recommend this one as its short and to the point:
<iframe width="560" height="315" src="https://www.youtube.com/embed/PNa9OMajw9w" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


The second one is slightly longer but is also an excellent talk held by [Bryan Hughes](https://github.com/nebrius)
<iframe width="560" height="315" src="https://www.youtube.com/embed/zphcsoSJMvM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


Now, relax, get a cup of tea and sit back while we go through everything together.

