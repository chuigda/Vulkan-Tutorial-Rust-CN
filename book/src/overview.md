# 概览

> 原文链接：<https://kylemayes.github.io/vulkanalia/overview.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

本章会以介绍 Vulkan 和它所解决的问题开始。之后，我们会看到绘制第一个三角形所需的所有组件。这会给你一个总体的蓝图，以便你将每个后续章节放在正确的位置。我们会通过 `vulkanalia` 实现的 Vulkan API 结构来总结。

## Vulkan 的起源

和之前的图形 API 一样，Vulkan 也是为跨平台抽象 [GPUs](https://en.wikipedia.org/wiki/Graphics_processing_unit) 而设计的。以往的 API 大都有一个问题，那就是它们都是根据诞生年代的图形硬件特性来设计的，而此时的图形硬件大多都只有一些可配置的功能。程序员必须以标准的格式提供顶点数据，并且在光照和着色选项上受制于 GPU 制造商。

在显卡架构成熟之后，它们开始提供更多的可编程特性。所有这些新功能都必须以某种方式与现有的 API 集成。这就导致这些 API 不能提供理想的抽象，而显卡驱动需要猜测程序员的意图，以将其映射到现代图形架构。这就是为什么有这么多驱动更新来提高游戏性能，而且有时候提升幅度很大。由于这些驱动的复杂性，应用程序开发人员还需要处理制造商之间的不一致性 例如 [着色器](https://en.wikipedia.org/wiki/Shader) 接受的语法。除了这些新功能之外，过去十年还涌入了具有强大图形硬件的移动设备。这些移动 GPU 出于空间和能耗上的考虑，采用了不同的架构。其中一个例子是 [tiled rendering](https://en.wikipedia.org/wiki/Tiled_rendering)，它可以给程序员提供对此功能的更多控制，从而提高性能。另一个起源于这些 API 时代的限制是有限的多线程支持，这可能会导致 CPU 成为性能瓶颈。

Vulkan 从头开始、针对现代图形架构而设计，从而解决了上述问题。Vulkan 要求程序员明确地指定他们的意图，从而减少驱动开销，并允许多个线程并行创建和提交命令。Vulkan 使用一种标准的字节码格式和一种编译器来减少着色器编译中的不一致性。最后，它将现代图形卡的通用处理能力纳入到单个 API 中，从而将图形和计算功能统一起来。

## 画一个三角形需要什么

接下来我们会总览一下在一个良好的 Vulkan 程序中绘制一个三角形所需的所有步骤。这里只是给你一个大的蓝图，以便你将所有的单独组件联系起来，而所有概念都会在后面的章节中详细介绍。

### 1. 创建实例并选择物理设备

一个 Vulkan 应用首先通过创建一个 `VkInstance` 来设置 Vulkan API。实例的创建是通过描述你的应用程序和你将要使用的 API 扩展来完成的。创建实例之后，你可以查询支持 Vulkan 的硬件，并选择一个或多个 `VkPhysicalDevice` 来使用。你可以查询像 VRAM 大小和设备功能这样的属性来选择所需的设备，例如优先使用独立显卡。

### 2. 逻辑设备和队列族（queue families）

选择正确的硬件设备后，你需要创建一个 `VkDevice` (逻辑设备)，在这里你需要更具体地描述你将要使用的 `VkPhysicalDeviceFeatures`，例如多视口渲染和 64 位浮点数。你还需要指定你想要使用的队列族。大多数 Vulkan 操作，例如绘制命令和内存操作，都是通过将它们提交到 `VkQueue` 来异步执行的。队列是从队列族中分配的，每个队列族都支持一组特定的操作。例如，可能会有单独的队列族用于图形、计算和内存传输操作。队列族的可用性也可以用作物理设备选择的区分因素。虽然支持 Vulkan 的设备可能不提供任何图形功能，但是今天所有支持 Vulkan 的显卡通常都支持我们感兴趣的所有队列操作。

### 3. 创建窗口和交换链（swapchain）

除非你只对离屏渲染有兴趣，否则你需要创建一个窗口来呈现渲染图像。窗口可以使用本地平台 API，或类似 [GLFW](http://www.glfw.org/)、[SDL](https://www.libsdl.org/) 或 [`winit`](https://github.com/rust-windowing/winit) crate。在本教程中我们会使用 `winit` crate，下一章会对其进行详细介绍。

我们还需要两个组件才能完成窗口渲染：一个窗口表面（`VkSurfaceKHR`）和一个交换链（`VkSwapchainKHR`），可以注意到这两个组件都有一个 `KHR` 后缀，这表示它们都是 Vulkan 扩展。Vulkan 本身完全是平台无关的，这就是为什么我们需要使用标准 WSI（Window System Interface，窗口系统接口）扩展与原生的窗口管理器进行交互。表面（Surface）是一个渲染窗口的跨平台抽象，通常它是由原生窗口系统句柄 —— 例如 Windows 上的 `HWND` —— 作为参数实例化得到的。然而，`vulkanalia` 包含了对 `winit` 可选的集成，这会帮助我们处理创建窗口和与之关联的表面的过程中那些平台特定的细节。

交换链是一系列的渲染目标。它可以保证我们正在渲染的图像不是当前屏幕上正在显然的图像，这样可以保证只有完整的图像才会被显示。每次我们想要绘制一帧时，我们都必须要求交换链提供一个图像来进行渲染。当我们完成一帧的绘制后，图像就会被返回到交换链中，以便在某个时刻呈现到屏幕上。渲染目标的数量和呈现图像到屏幕的条件取决于显示模式（present mode）。常见的显示模式有双缓冲（垂直同步）和三缓冲。我们将在创建交换链章节讨论这些问题。

 有的平台允许你直接渲染到输出，而不通过 `VK_KHR_display` 和 `VK_KHR_display_swapchain` 与窗口管理器进行交互。这就允许你创建一个覆盖整个屏幕的表面，你可以用它来实现你自己的窗口管理器。

### 4. 图像视图（image view）和帧缓冲（framebuffer）

从交换链获取图像后，还不能直接在图像上进行绘制，需要将图像先包装进 `VkImageView` 和 `VkFramebuffer`。一个图像试图可以引用图像的一个特定部分，而一个帧缓冲则引用了用于颜色、深度和模板的图像视图。因为交换链中可能有很多不同的图像，所以我们会预先为每个图像创建一个图像视图和帧缓冲，并在绘制时选择正确的那个。

### 5. 渲染流程（render passes）

Vulkan 中的渲染流程描述了渲染操作中使用的图像类型，图像的使用方式，以及如何处理它们的内容。在我们最初的三角形渲染程序中，我们会告诉 Vulkan 我们会使用一个图像作为颜色目标，并且我们希望在绘制操作之前将其清除为一个纯色。渲染流程只描述图像的类型，`VkFramebuffer` 则会将特定的图像绑定到这些槽中。

### 6. 图形管线（graphics pipeline）

Vulkan 的图形管线通过创建 `VkPipeline` 对象建立。它描述了显卡的可配置状态 —— 例如视口（viewport）的大小和深度缓冲操作，以及使用 `VkShaderModule 的可编程状态。`VkShaderModule` 对象是从着色器字节码创建的。驱动还需要知道在管线中将使用哪些渲染目标，我们通过引用渲染流程来指定。

Vulkan 与之前的图形 API 最大的不同是几乎所有图形管线的配置都需要提前完成。这也就意味着当我们想要切换到另一个着色器，或者稍微改变顶点布局，整个图形管线都要被重建。也就是说，我们需要为所有不同的组合创建很多 `VkPipeline` 对象。只有一些基本的配置，例如视口大小和清除颜色，可以动态改变。所有的状态都需要被显式地描述，没有默认的颜色混合状态。

这样做的好处类似于预编译相比于即时编译，驱动程序可以获得更大的优化空间，并且运行时的性能更加可预测，因为像切换到另一个图形管线这样的大的状态改变都是显式的。

### 7. 指令池和指令缓冲

之前提到，Vulkan 的许多操作 —— 例如绘制操作 —— 需要被提交到队列才能执行。这些操作首先要被记录到一个 `VkCommandBuffer` 中，然后提交给队列。这些指令缓冲由 `VkCommandPool` 分配，它与特定的队列族相关联。为了绘制一个简单的三角形，我们需要记录下列操作到 `VkCommandBuffer` 中：

* 开始渲染
* 绑定图形管线
* 绘制三个顶点
* 结束渲染

由于帧缓冲绑定的图像依赖于交换链给我们的图像，我们可以提前为每个图像创建指令缓冲，然后在绘制时直接选择对应的指令缓冲使用。当然，在每一帧都重新记录指令缓冲也是可以的，但这样做的效率很低。

### 8. 主循环

将绘制指令包装进指令缓冲之后，主循环就很直截了当了。我们首先使用 `vkAcquireNextImageKHR` 从交换链获取一张图像，接着为图像选择正确的指令缓冲，然后用 `vkQueueSubmit` 执行它。最后，我们使用 `vkQueuePresentKHR` 将图像返回到交换链，从而使其呈现到屏幕上。

提交给队列的操作会被异步执行。我们需要采取诸如信号量一类的同步措施来确保正确的执行顺序。绘制指令的执行必须是在获取图像完成后才能开始，否则可能会出现我们开始渲染到一个仍然在屏幕上显示的图像的情况。`vkQueuePresentKHR` 调用也需要等到渲染完成后才能执行，我们会使用第二个信号量来实现这一点。

### 总结

这个快速的介绍应该能让你对绘制第一个三角形所需的工作有一个基本的了解。一个真实的程序包含更多的步骤，例如分配顶点缓冲区、创建统一缓冲区和上传纹理图像，这些都会在后续章节中介绍，但我们会从简单的开始，因为 Vulkan 本身的学习曲线就已经非常陡峭了。请注意，我们会通过将顶点坐标嵌入到顶点着色器中来作弊，而不是使用顶点缓冲区。这是因为管理顶点缓冲区需要对命令缓冲区有一定的了解。

所以简单来说，要绘制第一个三角形，我们需要：

* 创建一个 `VkInstance`
* 选择一个支持的显卡（`VkPhysicalDevice`）
* 创建用于绘制和呈现的 `VkDevice` 和 `VkQueue`
* 创建窗口、窗口表面和交换链
* 将交换链图像包装进 `VkImageView`
* 创建描述渲染目标和用途的渲染流程
* 为渲染流程创建帧缓冲
* 设置图形管线
* 为每个可能的交换链图像分配并记录一个包含绘制指令的指令缓冲
* 通过获取图像、提交正确的绘制指令缓冲，然后将图像返回到交换链来绘制帧

步骤非常多，但其实每一步都非常简单。这些步骤中的每一个都会在后续章节中详细介绍。如果你对程序中的某一步感到困惑，可以回来参考一下本章节。

## API 概念

Vulkan API 是用 C 语言定义的。Vulkan API 的规范 —— Vulkan API 注册表 —— 是用 [一个 XML 文件](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/xml/vk.xml) 来定义的，它提供了机器可读的 Vulkan API 定义。

[Vulkan 头文件](https://github.com/KhronosGroup/Vulkan-Headers) 是 Vulkan SDK 的一部分，它们是从 Vulkan API 注册表生成的。在下一章中我们将安装的 Vulkan SDK 包含了这些头文件。然而，我们不会直接或间接地使用这些头文件，因为 `vulkanalia` 包含了一个独立于 Vulkan SDK 提供的 C 接口的 Rust 接口，这个接口也是从 Vulkan API 注册表生成的。

`vulkanalia` 的基础是 [`vulkanalia-sys`](https://docs.rs/vulkanalia-sys) crate，它定义了 Vulkan API 注册表中的原始类型。这些原始类型被 `vulkanalia` crate 在 [`vk`](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/vk/index.html) 模块中重新导出，同时还包含了从 Vulkan API 注册表生成的其他一些项目，作为前面介绍中提到的对 Vulkan API 的轻量级包装。

### 类型名称

因为 Rust 有对名称空间（namespace）的支持而 C 没有，`vulkanalia` 的 API 会略去 Vulkan 类型名称中用于命名空间的部分。更具体地说，Vulkan 类型，例如结构体、联合和枚举，没有 `Vk` 前缀。例如，`VkInstanceCreateInfo` 结构体在 `vulkanalia` 中变成了 [`InstanceCreateInfo`](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/vk/struct.InstanceCreateInfo.html) 结构体，并且可以在前面提到的 [`vk`](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/vk/index.html) 模块中找到。

从现在开始，本教程将使用 `vulkanalia` 中的 `vk::` 模块前缀来引用 `vulkanalia` 中定义的 Vulkan 类型，以明确该类型表示的是从 Vulkan API 注册表生成的东西。

这些类型名称会被链接到 `vulkanalia` 文档中对应的类型。Vulkan 类型的 `vulkanalia` 文档还包含一个指向 Vulkan规范中该类型的链接，可以用来了解该类型的目的和用法。

一些类型名的例子：

* `vk::Instance`&nbsp;
* `vk::InstanceCreateInfo`&nbsp;
* `vk::InstanceCreateFlags`&nbsp;

### 枚举

`vulkanalia` 将 Vulkan 枚举实现为结构体，并将枚举变体实现为这些结构体的关联常量。不使用 Rust 枚举是因为在 FFI 调用中使用 Rust 枚举可能导致 [未定义行为](https://github.com/rust-lang/rust/issues/36927)。

因为结构体充当了关联常量的名称空间，我们也就不必担心不同 Vulkan 枚举（或来自其他库的枚举）名称之间的冲突，就像在 C 中那样。所以，就像类型名称一样，`vulkanalia` 会略去 Vulkan 枚举名称中用于名称空间的部分。

例如，`VK_OBJECT_TYPE_INSTANCE` 枚举变体是 `VkObjectType` 枚举的 `INSTANCE` 值。在 `vulkanalia` 中，这个变体变成了 `vk::ObjectType::INSTANCE`。

### 掩码（bitmasks）

`vulkanalia` 将掩码实现为结构体，并将位标志（bitflags）实现为这些结构体的关联常量。这些结构体和关联常量是通过由 [`bitflags`](https://github.com/bitflags/bitflags) crate 提供的 `bitflags!` 宏来生成的。

和枚举变体一样，位标志名中用于名称空间的部分会被略去。

例如，`VK_BUFFER_USAGE_TRANSFER_SRC_BIT` 位标志是 `VkBufferUsageFlags` 掩码的 `TRANSFER_SRC` 位标志。在 `vulkanalia` 中，这个位标志变成了 `vk::BufferUsageFlags::TRANSFER_SRC`。

### 命令

诸如 `vkCreateInstance` 的原始 Vulkan 命令的类型在 `vulkanalia` 中被定义为带有 `PFN_`（pointer to function，函数指针）前缀的函数指针类型别名。所以 `vkCreateInstance` 的 `vulkanalia` 类型别名是 `vk::PFN_vkCreateInstance`。

这些函数签名本身还不足以调用 Vulkan 命令，我们必须先加载这些类型所描述的命令。Vulkan 规范针对这个问题有一个[详细的描述](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#initialization-functionpointers)，但是在这里我会给出一个简化的版本。

第一个要加载的命令是 `vkGetInstanceProcAddr`，这个命令是以平台特定的方式加载的，但是 `vulkanalia` 提供了一个可选的 [`libloading`](https://crates.io/crates/libloading) 集成，我们会在本教程中使用它来从 Vulkan 共享库中加载这个命令。`vkGetInstanceProcAddr` 可以用来加载我们想要调用的其他 Vulkan 命令。

然而，取决于系统上的 Vulkan 实现，可能会有多个版本的 Vulkan 命令可用。例如，如果你的系统上有一个独立的 NVIDIA GPU 和一个集成的 Intel GPU，那么可能会有针对每个设备的专用 Vulkan 命令的不同实现，例如 `allocate_memory`。在这种情况下，`vkGetInstanceProcAddr` 会返回一个命令，这个命令会根据使用的设备来分派调用到正确的设备特定命令。

要避免这种分派的运行时开销，可以使用 `vkGetDeviceProcAddr` 命令来直接加载这些设备特定的 Vulkan 命令。这个命令的加载方式和 `vkGetInstanceProcAddr` 一样。

我们会在这个教程中用到许多 Vulkan 命令。幸运的是，我们不需要手动加载它们，因为 `vulkanalia` 已经提供了以下四类结构体，可以用来轻松地加载所有 Vulkan 命令：

* `vk::StaticCommands` &ndash; 以平台特定的方式加载的 Vulkan 命令，可以用来加载其他命令（例如 `vkGetInstanceProcAddr` 和 `vkGetDeviceProcAddr`）
* `vk::EntryCommands` &ndash; 使用 `vkGetInstanceProcAddr` 和一个空的 Vulkan 实例加载的 Vulkan 命令。这些命令不与特定的 Vulkan 实例绑定，可以用来查询实例支持并创建实例
* `vk::InstanceCommands` &ndash; 使用 `vkGetInstanceProcAddr` 和一个有效的 Vulkan 实例加载的 Vulkan 命令。这些命令与特定的 Vulkan 实例绑定，可以用来查询设备支持并创建设备
* `vk::DeviceCommands` &ndash; 使用 `vkGetDeviceProcAddr` 和一个有效的 Vulkan 设备加载的 Vulkan 命令。这些命令与特定的 Vulkan 设备绑定，并且提供了你期望中图形 API 提供的大多数功能

这些结构体能让你简单地在 Rust 中加载和调用原始 Vulkan 命令，不过 `vulkanalia` 提供了对原始命令的包装，这使得在 Rust 中使用它们更加容易，并且不易出错。

### 命令包装器（wrapper）

一个典型的 Vulkan 命令的签名在 C 中看起来就像这样：

```c
VkResult vkEnumerateInstanceExtensionProperties(
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties
);
```

熟悉 Vulkan API 的人可以从这个签名中快速看出这个命令的用法，尽管它没有包含一些关键信息。

而对于那些刚接触 Vulkan API 的人来说，查看此命令的[文档](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkEnumerateInstanceExtensionProperties.html)可能会更有启发性。文档中对此命令行为的描述表明，使用此命令列出 Vulkan 实例可用的扩展（extension）需要多个步骤：

1. 调用命令以获取扩展的数量
2. 分配一个可以容纳输出的缓冲区
3. 再次调用命令，获取扩展并填充缓冲区

所以在 C++ 中，这些步骤可能看起来像这样（简单起见，这里忽略了命令的结果）：

```c++
// 1.
uint32_t pPropertyCount;
vkEnumerateInstanceExtensionProperties(NULL, &pPropertyCount, NULL);

// 2.
std::vector<VkExtensionProperties> pProperties{pPropertyCount};

// 3.
vkEnumerateInstanceExtensionProperties(NULL, &pPropertyCount, pProperties.data());
```

而 `vkEnumerateInstanceExtensionProperties` 的包装器的 Rust 签名如下：

```rust,noplaypen
unsafe fn enumerate_instance_extension_properties(
    &self,
    layer_name: Option<&[u8]>,
) -> VkResult<Vec<ExtensionProperties>>;
```

这个命令包装器使得从 Rust 使用 `vkEnumerateInstanceExtensionProperties` 更加容易、更少出错，并且更符合惯用法：

* `layer_name` 参数的可选性被编码在函数签名中。这个参数是可选的，这一点在 C 函数签名中没有体现，需要查阅 Vulkan 规范才能得到这个信息
* 命令的可失败性通过返回一个 `Result`（[`VkResult<T>`](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/type.VkResult.html) 是 `Result<T, vk::ErrorCode>` 的类型别名）体现。这使得我们可以利用 Rust 强大的错误处理能力，并且在我们忽略检查可失败命令的结果时，编译器会发出警告
* 命令包装器在内部处理了上面描述的三个步骤，并返回一个包含扩展属性的 `Vec`

注意，命令包装器仍然是 `unsafe` 的，因为虽然 `vulkanalia` 可以消除某些类型的错误（例如给此命令传递一个空的层名称，）但还是有很多可能会出错的事情，导致诸如段错误之类“有趣”的事情发生。你可以随时检查 Vulkan 文档中命令的 `Valid Usage` 部分以了解如何正确地调用命令。

你可能注意到了上面命令包装器中的 `&self` 参数。这些命令包装器是在 trait 中定义的，而 `vulkanalia` 暴露的类型实现了这些 trait。这些 trait 可以分为两类：版本 trait （version traits）和扩展 trait（extension traits）。版本 trait 为 Vulkan 的标准部分中的命令提供命令包装器，而扩展 trait 为 Vulkan 扩展中的命令提供命令包装器。

例如，`enumerate_instance_extension_properties` 是一个非扩展 Vulkan 命令，是 Vulkan 1.0 的一部分，不依赖于 Vulkan 实例或设备，所以它被放在 `vk::EntryV1_0` trait 中。而 `cmd_draw_indirect_count` 命令是在 Vulkan 1.2 中添加的，并且依赖于 Vulkan 设备，所以它被放在 `vk::DeviceV1_2` trait 中。

而 `vk::KhrSurfaceExtension` 是一个扩展 trait，我们将在后面的章节中使用它来调用 `destroy_surface_khr` 这样的 Vulkan 命令，这些命令是在 `VK_KHR_surface` 扩展中定义的。

<!-- TODO: needs much refinement -->
这些版本和扩展 trait 是为包含加载的命令和所需的 Vulkan 实例或设备（如果有的话）的类型定义的。这些类型是精心手工制作的，而不是 `vulkanalia` 的 `vk` 模块中自动生成的 Vulkan 绑定的一部分。它们是 `Entry`、`Instance` 和 `Device` 结构体，将在后面的章节中使用。

从现在开始，本教程将继续像本章节一样直接按名称引用这些命令包装器（例如 `create_instance`）。你可以访问 `vulkanalia` 文档来获取命令包装器的更多信息，例如命令包装器是在哪个 trait 中定义的。

<!--
1. 机械工业出版社翻译的《GoF 设计模式》使用了“生成器模式”
2. Wikipedia 也使用了“生成器模式”: https://zh.wikipedia.org/zh-cn/%E7%94%9F%E6%88%90%E5%99%A8%E6%A8%A1%E5%BC%8F
3. Rust 的生成器 (generator) 还早得很，<i>而且我觉得那玩意没屌用</i>，不用管它
-->
### 生成器（Builders）

Vulkan API 通常使用结构体作为 Vulkan 命令的参数。这些作为命令的参数使用的 Vulkan 结构体有一个字段，用于指示结构体的类型。在 C API 中，这个字段（`sType`）需要被显式地设置。例如，这里我们正在填充 `VkInstanceCreateInfo` 的一个实例，然后在 C++ 中使用它来调用 `vkCreateInstance`：

```c++
std::vector<const char*> extensions{/* 3 extension names */};

VkInstanceCreateInfo info;
info.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
info.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
info.ppEnabledExtensionNames = extensions.data();

VkInstance instance;
vkCreateInstance(&info, NULL, &instance);
```

当使用 `vulkanalia` 时，你仍然可以用这种方式填充参数结构体，但是 `vulkanalia` 提供了生成器（builder），简化了这些参数结构体的构造。在 `vulkanalia` 中，`vk::InstanceCreateInfo` 对应的生成器是 `vk::InstanceCreateInfoBuilder`。使用这个生成器，上面的代码就可以写成：

```rust,noplaypen
let extensions = &[/* 3 extension names */];

let info = vk::InstanceCreateInfo::builder()
    .enabled_extension_names(extensions)
    .build();

let instance = entry.create_instance(&info, None).unwrap();
```

注意以下差异：

* 无需为 `s_type` 字段提供值。这是因为生成器会自动为这个字段提供正确的值（`vk::StructureType::INSTANCE_CREATE_INFO`）
* 无需为 `enabled_extension_count` 字段提供值。这是因为生成器的 `enabled_extension_names` 方法会自动使用提供的切片的长度设置这个字段

<!-- 
生命周期属于是误译，显然 Rust 的 lifetime 里面没有“周”。
生命周期应该用来指 React，Vue 的 lifecycle 那种东西。
所以我们选择了“生存期”。
-->
然而，上面的 Rust 代码有一定程度的危险。生成器有生存期（lifetime），这要求生成器中存储的引用至少要与生成器本身活得一样久。也就是说，在上面的例子中，Rust 编译器会确保传递给 `enabled_extension_names` 方法的切片至少活得与生成器一样长。然而，一旦我们调用 `.build()` 来获取底层的 `vk::InstanceCreateInfo` 结构体，生成器的生存期就会被丢弃。这意味着 Rust 编译器不再能防止我们 _搬起石头砸自己的脚_，例如解引用一个已经不存在的切片的指针。

下面的代码会崩溃（但愿如此），因为传递给 `enabled_extension_names` 的临时 `Vec` 在我们使用 `vk::InstanceCreateInfo` 结构体调用 `create_instance` 时已经被丢弃了：

```rust,noplaypen,panics
let info = vk::InstanceCreateInfo::builder()
    .enabled_extension_names(&vec![/* 3 extension names */])
    .build();

let instance = entry.create_instance(&info, None).unwrap();
```

幸运的是，`vulkanalia` 为此提供了解决方案 —— 不调用 `build()`，而是直接将生成器传递给命令包装器！在任何接受 Vulkan 结构体的地方，你都可以直接提供与 Vulkan 结构体对应的生成器。如果从上面的代码中删除 `build()` 调用，Rust 编译器就能够利用生成器上的生存期来拒绝这个坏代码，并告诉你 `error[E0716]: temporary value dropped while borrowed`。

### `prelude` 模块

`vulkanalia` 提供了[`prelude` 模块](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/prelude/index.html)，用于暴露使用 crate 所需的基本类型。每个 Vulkan 版本都有一个 `prelude` 模块，每个模块都会暴露相关的命令 trait，以及其他经常用到的类型：

```rust,noplaypen
// Vulkan 1.0
use vulkanalia::prelude::v1_0::*;

// Vulkan 1.1
use vulkanalia::prelude::v1_1::*;

// Vulkan 1.2
use vulkanalia::prelude::v1_2::*;
```

<!-- VulkanTutorialCN 使用了这个翻译 -->
## 校验层（Validation layers）

如前文所述，Vulkan 是为高性能和低驱动程序开销而设计的。因此，默认情况下 Vulkan 只包含非常有限的错误检查和调试功能。如果你做错了什么，驱动程序通常会崩溃而不是返回错误代码，或者比这更糟 —— 程序会在你的显卡上运行，但在其他显卡上完全失效。

你可以通过*校验层*来在 Vulkan 中启用很多检查。校验层是可以插入到 API 和图形驱动程序之间的代码片段，用于对函数参数进行额外的检查，并且跟踪内存管理问题。你可以在开发时启用它们，然后在发布应用程序时将其完全禁用，从而实现零开销。任何人都可以编写自己的校验层，但是 LunarG 的 Vulkan SDK 提供了一套标准的校验层，我们将在本教程中使用它们。你还需要注册一个回调函数来接收校验层的调试消息。

因为 Vulkan 对每个操作都非常明确，校验层也非常广泛，所以实际上相比于 OpenGL 和 Direct3D，你更容易找出为什么你的画面是全黑的！
