---
title: "理解浏览器的多进程架构"
date: 2024-03-02
---

由于 Chrome 是目前最流行的浏览器，因此本文实际上讨论的是关于 Chrome 的多进程架构。

## 进程和线程

在了解浏览器的架构之前，我们首先需要掌握[进程（Process](<https://en.wikipedia.org/wiki/Process_(computing)>)）和[线程（Thread）](<https://en.wikipedia.org/wiki/Thread_(computing)>)这两个常见的计算机术语。

计算机系统中的进程是对 CPU、内存和 I/O 设备等硬件的一种抽象。简单说，进程就是操作系统正在运行的程序。操作系统可以同时运行多个进程，而每个进程也可以由多个称为线程的执行单元组成。线程运行在进程的上下文中，并共享同样的代码和全局数据。

每当启动一个应用程序时（比如 Chrome），系统会自动创建多个进程，每个进程还会在内部创建多个线程来协同工作。操作系统为进程分配一块可用的内存，应用程序的所有状态都保存在这块私有的内存空间中。关闭应用程序后，进程也随之消失，被进程占用的内存由操作系统进行回收。

Windows 可以在 Task Manager 中查看当前运行的所有进程，有时候某个软件崩溃了，我们也可以在这里找到对应的进程并强制关闭它。

进程之间使用进程间通信（[Inter Process Communication, IPC](https://en.wikipedia.org/wiki/Inter-process_communication)）进行交流。

## 为什么需要多进程架构？

很早之前，浏览器中的标签页、渲染引擎或插件都运行在同一个进程中，只要其中一个发生错误就会导致整个浏览器崩溃。

为了解决这个问题，Chrome 引入了[多进程架构（Multi-Process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)），并且正在往[面向服务的架构（Service-Oriented Architecture）](https://www.chromium.org/servicification)上进行变革。

这意味着，即使 Chrome 只打开一个标签页，它也会创建多个进程，每个进程拥有唯一的 Process ID (PID):

- **Browser Process**：主要负责控制 Chrome 的地址栏、书签、前进和后退按钮，以及网络请求和文件访问。
- **Renderer Process**：核心任务是将 HTML、CSS 和 JS 转换为可交互的网页，换句话说，也就是负责页面渲染、处理事件和执行脚本。开发者最为熟悉的 [Blink (Rendering engine)](https://www.chromium.org/blink/) 和 [V8 (JavaScript engine)](https://v8.dev/) 都运行在该进程中。默认情况下，Chrome 为每个标签页都创建了一个渲染进程，并运行在沙箱模式下，相互隔离。
- **Plugin Process**：控制浏览器插件的运行，每个插件都有一个单独的插件进程，即使插件崩溃也不会对浏览器、标签页或其他插件产生影响。
- **GPU Process**：负责高效地渲染网页上的视频和图形，以提高用户浏览体验。

多进程架构的优点有很多：

- 稳定性。由于进程之间相互隔离，即使某个标签页或 Plugin Process 失去响应，也不会影响到其他的进程或浏览器的正常运行；
- 安全性。浏览器可以使用沙箱将某些进程锁起来，限制其获取系统权限的能力；
- 多进程可以充分利用现代 CPU 多核的优势，提高浏览器的性能。

## 面向服务的架构

另一方面来说，多进程架构的缺点是内存占用过高和日益增加的复杂性。为此，Chrome 将原来的代码迁移到了更加模块化和[面向服务的架构（Service-Oriented Architecture）](https://www.chromium.org/servicification)。这种新的架构将产生更多可重用和解耦的组件，当然也意味着减少了重复，而且还能让开发者团队在不修改 Chrome 的前提下实验新功能。

当 Chrome 运行在高性能的设备上时，会将多个基础服务分配到不同的进程中，从而提升稳定性。但是如果运行在内存受限的设备上时，则会将多个基础服务聚合到一个进程中，从而减少内存的占用。

## References

- [Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)
