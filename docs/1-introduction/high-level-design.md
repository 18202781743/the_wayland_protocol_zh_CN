---
outline: [2,3]
---

# Wayland 的上层设计

您的电脑有“输入”和“输出”设备，分别负责接收您的信息和显示信息。输入设备通常有以下几种：

- 键盘
- 鼠标
- 触控板
- 触摸屏
- 数位板

输出设备一般是桌面上的显示器、笔记本或其他移动设备的屏幕。应用程序之间将会共享这些屏幕，而“Wayland 合成器”的作用便是分配输入事件到正确的“Wayland 客户端”，并将他们的程序窗口恰当地显示在您的显示器上。

将所有应用程序的窗口组合起来，一起显示在屏幕上的过程被称为“合成”，而执行这一过程的程序被称作“合成器”。

::: tip

Wayland Compositor，“合成器”、“混成器”只是不同翻译

:::


## 具体实践

桌面生态系统包含有许多程序组件，比如渲染用的 Mesa（及其驱动）、Linux KMS/DRM 子系统、负责缓冲区分配的 GBM、用户态的 libdrm、libinput、evdev 等。不用担心，理解 Wayland 几乎不需要这些专业知识，何况这些内容都远超本书范围。实际上，Wayland 协议是相当保守和高度封装的，很容易就能构建出一个 Wayland 桌面并成功运行多数应用程序，无需担心与不同应用程序的具体纠缠。话虽如此，浅浅了解这些东西，它们是什么以及它们如何工作，仍非常有用。让我们自下而上展开。

## 硬件部分

一台典型的计算机配备了一些重要的硬件。在机箱外面，我们有显示器、键盘、鼠标，或许还有麦克风和一个可爱的 USB 加热杯垫。机箱内部有一系列组件与这些设备接口相连。例如，如果您的电脑有 USB 接口 - 那您的键鼠大概率会插在 USB 接口上，显示器正连接着显卡 GPU。

这些硬件各司其职。例如，GPU 有以显存形式提供的像素缓冲区，并将这些像素扫描输出到显示器上。GPU 还提供经过特调整的处理器，它们虽然在其他方面有所不如，但可以很好地处理高度并行的任务（例如为 1080P 显示器上的 2,073,600 个像素计算正确的颜色）。USB 控制器的工作同样复杂的令人称奇，它要实现枯燥的 USB 协议，接受来自键盘的输入事件，或精心调控杯垫的温度，从而避免投诉和令人不快的冷咖啡。

在这个层面上，硬件几乎不了解系统上正在运行着哪些应用程序。硬件提供了执行工作的指令接口，并被告知相应的操作的指令，但不在乎指令是谁发出的。基于此，只有一个组件被允许和硬件交流。

## 内核部分

这一任务落归内核组件。内核是一头复杂的“野兽”，因此我们只关注与 Wayland 相关的部分。Linux 内核的任务是给硬件提供一个抽象，这样在用户态可以安全的访问它们，我们的 Wayland 合成器也运行在用户态。对于图形子系统 DRM（direct rendering manager）来说，就可以在用户态有效地给 GPU 分配任务。另一个重要的子系统是 KMS（kernel mode setting），用于枚举显示设备并设置其属性，比如选定的分辨率（也称为“模式”）等。输入设备通过名为 evdev 的接口进行抽象。

大多数内核接口都以特殊文件的形式存在于 `/dev` 供用户态调用。以 DRM 为例，这些文件位于 `/dev/dri/`，通常，以主要节点 primary node（如 card0）的形式进行模式设置等特权操作，以渲染节点 render node（如 renderD128）的形式进行渲染、视频解码等非特权操作，而对于设备节点 device nodes 则位于 `/dev/input/event*`

```shell
$ ls /dev/dri/
by-path  card0  renderD128
```

## 用户态

现在我们来看用户态。在这里，应用程序与硬件隔离，必须通过内核提供的设备节点 device nodes 才能运行。

### libdrm

大多数 Linux 内核接口都有一个对应的用户态，它为使用这些设备节点提供了令人满意的 C 语言 API。libdrm 库是其中之一，它是 DRM 子系统的用户态部分。Wayland 混成器使用它进行模式设置和其他 DRM 操作，但 Wayland 客户端通常不直接使用 libdrm。

### Mesa

Mesa 是 Linux 图形栈中最为重要的部分之一。它除了为 Linux 提供 OpenGL（和 Vulkan）的厂家优化实现之外，还提供了 GBM（Generic Buffer Management）库，这是一种在 libdrm 之上的抽象层，用于在 GPU 上分配缓冲区。大多数 Wayland 混成器将通过 Mesa 同时使用 GBM 和 OpenGL，多数客户端至少使用 OpenGL 或 Vulkan 其中一种。

### libinput

如同 libdrm 是 DRM 子系统的抽象那样，libinput 提供了 evdev 用户态的抽象。它负责从内核接收输入设备的输入事件，将其解码为可用的形式，并传递给 Wayland 混成器。混成器需要特殊的权限才能使用 evdev 设备文件，从而迫使 Wayland 客户端通过混成器接收输入事件，这样可以防止键盘被记录日志等。

### (e)udev

用户态还要负责协调来自内核的新设备、配置 `/dev` 的设备节点的权限、并将事件变动发送给系统上正在运行的程序。大多数系统使用 udev（或 eudev）以进行这项工作。Wayland 就是用 udev 来枚举输入设备和 GPU，并在出现变动时接收通知。

### xkbcommon

XKB（X Keyboard）最初是 Xorg 服务处理键盘的子系统。几年前开发者将它从 Xorg 代码中分离出来，成为了一个独立的键盘处理库，从此不再与 X 有任何实际的联系。Libinput（以及 Wayland 混成器）以扫描码的形式提供键盘事件，扫描码的准确含义因键盘而异。xkbcommon 负责将这些扫描码转化为更有意义的通用键盘信号，如 `65` 转化为 `XKB_KEY_Space`。它还包含了一个状态机，负责将在按住 shift 键的时的 `1` 会变成 `！`。

### pixman

这是一个简单的、客户端和混成器都使用的像素操作（pixel manipulation）库，它可以有效的处理像素缓冲区、处理图像矩形的数学运算，以及执行其他相关的像素操作任务。

### libwayland

libwayland 是 Wayland 协议最常用的 C 语言实现，能处理大部分的底层传输协议。libwayland 同时也提供了一个从 Wayland 协议（XML 文件）生成高级代码的工具。我们将从第 1.3 章开始，在本书中详细讨论 libwayland。

### 其他

到目前为止，提到的每个部分在整个 Linux 桌面生态系统中都是一致的，除此以外还存在更多的组件。许多图形应用程序根本不知晓 Wayland，而是选择诸如 GTK、QT、SDL 和 GLFW 之类的库来进行代处理。许多混成器选择像 wlroots 这样的软件来抽象简化很多功能，而其它的一类混成器则在内部实现所有功能。