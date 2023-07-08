# 概览

> 原文链接：<https://kylemayes.github.io/vulkanalia/overview.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

本章会以介绍 Vulkan 和它所解决的问题开始。之后，我们会看到绘制第一个三角形所需的所有组件。这会给你一个总体的蓝图，以便你将每个后续章节放在正确的位置。我们会通过 `vulkanalia` 实现的 Vulkan API 结构来总结。

## Vulkan 的起源

和之前的图形 API 一样，Vulkan 也是为跨平台抽象 [GPUs](https://en.wikipedia.org/wiki/Graphics_processing_unit) 而设计的。以往的 API 大都有一个问题，那就是在它们诞生的年代，它们是根据图形硬件的特性来设计的，而此时的图形硬件大多都只有一些可配置的功能。程序员必须以标准的格式提供顶点数据，并且在光照和着色选项上受制于 GPU 制造商。

在显卡架构成熟之后，它们开始提供更多的可编程特性。所有这些新功能都必须以某种方式与现有的 API 集成。这就导致这些 API 不能提供理想的抽象，而显卡驱动需要猜测程序员的意图，以将其映射到现代图形架构。这就是为什么有这么多驱动更新来提高游戏性能，而有时候提升幅度很大。由于这些驱动的复杂性，应用程序开发人员还需要处理制造商之间的不一致性 例如 [着色器](https://en.wikipedia.org/wiki/Shader) 接受的语法。除了这些新功能之外，过去十年还涌入了具有强大图形硬件的移动设备。这些移动 GPU 出于空间和能耗上的考虑，采用了不同的架构。其中一个例子是 [tiled rendering](https://en.wikipedia.org/wiki/Tiled_rendering)，它可以给程序员提供对此功能的更多控制，从而提高性能。另一个起源于这些 API 时代的限制是有限的多线程支持，这可能会导致 CPU 成为性能瓶颈。

Vulkan 从头开始、针对现代图形架构而设计，从而解决了这些问题。Vulkan 要求程序员明确地指定他们的意图，从而减少驱动开销，并允许多个线程并行创建和提交命令。Vulkan 使用一种标准的字节码格式和一种编译器来减少着色器编译中的不一致性。最后，它将现代图形卡的通用处理能力纳入到单个 API 中，从而将图形和计算功能统一起来。

## 画一个三角形需要什么

接下来我们会总览一下在一个良好的 Vulkan 程序中绘制一个三角形所需的所有步骤。这里介绍的所有概念都会在后面的章节中详细介绍。这里只是给你一个大的蓝图，以便你将所有的单独组件联系起来。

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

因为结构体充当了关联常量的命名空间，我们也就不必担心不同 Vulkan 枚举（或来自其他库的枚举）名称之间的冲突，就像在 C 中那样。所以，就像类型名称一样，`vulkanalia` 会略去 Vulkan 枚举名称中用于命名空间的部分。

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

### 命令封装

一个典型的 Vulkan 命令的签名在 C 中看起来就像这样：

```c
VkResult vkEnumerateInstanceExtensionProperties(
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties
);
```

熟悉 Vulkan API 的人可以从这个签名中快速看出这个命令的用法，尽管它没有包含一些关键信息。

For those new to the Vulkan API, a look at the [documentation](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkEnumerateInstanceExtensionProperties.html) for this command will likely be more illuminating. The description of the behavior of this command in the documentation suggests that using this command to list the available extensions for the Vulkan instance will be a multi-step process:

 1. Call the command to get the number of extensions
 2. Allocate a buffer that can contain the outputted number of extensions
 3. Call the command again to populate the buffer with the extensions

So in C++, this might look like this (ignoring the result of the command for simplicity):

```c++
// 1.
uint32_t pPropertyCount;
vkEnumerateInstanceExtensionProperties(NULL, &pPropertyCount, NULL);

// 2.
std::vector<VkExtensionProperties> pProperties{pPropertyCount};

// 3.
vkEnumerateInstanceExtensionProperties(NULL, &pPropertyCount, pProperties.data());
```

The Rust signature of the wrapper for `vkEnumerateInstanceExtensionProperties` looks like this:

```rust,noplaypen
unsafe fn enumerate_instance_extension_properties(
    &self,
    layer_name: Option<&[u8]>,
) -> VkResult<Vec<ExtensionProperties>>;
```

This command wrapper makes the usage of `vkEnumerateInstanceExtensionProperties` from Rust easier, less error-prone, and more idiomatic in several ways:

* The optionality of the `layer_name` parameter is encoded in the function signature. That this parameter is optional is not captured in the C function signature, one would need to check the Vulkan specification for this information
* The fallibility of the command is modelled by returning a `Result` ([`VkResult<T>`](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/type.VkResult.html) is a type alias for `Result<T, vk::ErrorCode>`). This allows us to take advantage of Rust's strong error handling capabilities as well as be warned by the compiler if we neglect to check the result of a fallible command
* The command wrapper handles the three step process described above internally and returns a `Vec` containing the extension properties

Note that command wrappers are still `unsafe` because while `vulkanalia` can eliminate certain classes of errors (e.g., passing a null layer name to this command), there are still plenty of things that can go horribly wrong and cause fun things like segfaults. You can always check the `Valid Usage` section of the Vulkan documentation for a command to see the invariants that need to upheld to call that command validly.

You likely noticed the `&self` parameter in the above command wrapper. These command wrappers are defined in traits which are implemented for types exposed by `vulkanalia`. These traits can be separated into two categories: version traits and extension traits. The version traits offer command wrappers for the commands which are a standard part of Vulkan whereas the extension traits offer command wrappers for the commands which are defined as part of Vulkan extensions.

For example, `enumerate_instance_extension_properties` is in the `vk::EntryV1_0` trait since it is a non-extension Vulkan command that is part of Vulkan 1.0 and not dependent on a Vulkan instance or device. A Vulkan command like `cmd_draw_indirect_count` that was added in Vulkan 1.2 and is dependent on a Vulkan device would be in the `vk::DeviceV1_2` trait.

`vk::KhrSurfaceExtension` is an example of an extension trait that we will be using in future chapters to call Vulkan commands like `destroy_surface_khr` that are defined in the `VK_KHR_surface` extension.

These version and extension traits are defined for types which contain both the loaded commands and the required Vulkan instance or device (if any). These types have been lovingly hand-crafted and are not part of the generated Vulkan bindings in the `vk` module of `vulkanalia`. They will be used in future chapters and are the `Entry`, `Instance`, and `Device` structs.

Going forward, this tutorial will continue to refer to these command wrappers directly by name as in this section (e.g., `create_instance`). You can visit the `vulkanalia` documentation for the command wrapper for more information like which trait the command wrapper is defined in.

### Builders

The Vulkan API heavily utilizes structs as parameters for Vulkan commands. The Vulkan structs used as command parameters have a field which indicates the type of the struct. In the C API, this field (`sType`) would need to be set explicitly. For example, here we are populating an instance of `VkInstanceCreateInfo` and then using it to call `vkCreateInstance` in C++:

```c++
std::vector<const char*> extensions{/* 3 extension names */};

VkInstanceCreateInfo info;
info.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
info.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
info.ppEnabledExtensionNames = extensions.data();

VkInstance instance;
vkCreateInstance(&info, NULL, &instance);
```

You can still populate parameter structs in this manner when using `vulkanalia`, but `vulkanalia` provides builders which simplify the construction of these parameter structs. The `vulkanalia` builder for `vk::InstanceCreateInfo` is `vk::InstanceCreateInfoBuilder`. Using this builder the above code would become:

```rust,noplaypen
let extensions = &[/* 3 extension names */];

let info = vk::InstanceCreateInfo::builder()
    .enabled_extension_names(extensions)
    .build();

let instance = entry.create_instance(&info, None).unwrap();
```

Note the following differences:

* A value is not provided for the `s_type` field. This is because the builder provides the correct value for this field (`vk::StructureType::INSTANCE_CREATE_INFO`) automatically
* A value is not provided for the `enabled_extension_count` field. This is because the `enabled_extension_names` builder method uses the length of the provided slice to set this field automatically

However, the above Rust code involves a certain degree of danger. The builders have lifetimes which enforce that the references stored in them live at least as long as the builders themselves. In the above example, this means that the Rust compiler will make sure that the slice passed to the `enabled_extension_names` method lives at least as long as the builder. However, as soon as we call `.build()` to get the underlying `vk::InstanceCreateInfo` struct the builder lifetimes are discarded. This means that the Rust compiler can no longer prevent us from shooting ourselves in the foot if we try to dereference a pointer to a slice that no longer exists.

The following code will (hopefully) crash since the temporary `Vec` passed to `enabled_extension_names` will have been dropped by the time we call `create_instance` with our `vk::InstanceCreateInfo` struct:

```rust,noplaypen
let info = vk::InstanceCreateInfo::builder()
    .enabled_extension_names(&vec![/* 3 extension names */])
    .build();

let instance = entry.create_instance(&info, None).unwrap();
```

Fortunately, `vulkanalia` has a solution for this. Simply don't call `build()` and instead pass the builder to the command wrapper instead! Anywhere a Vulkan struct is expected in a command wrapper you can instead provide the associated builder. If you remove the `build()` call from the above code the Rust compiler will be able to use the lifetimes on the builder to reject this bad code with `error[E0716]: temporary value dropped while borrowed`.

### Preludes

`vulkanalia` offers [prelude modules](https://docs.rs/vulkanalia/%VERSION%/vulkanalia/prelude/index.html) that expose the basic types needed to use the crate. One prelude module is available per Vulkan version and each will expose the relevant command traits along with other very frequently used types:

```rust,noplaypen
// Vulkan 1.0
use vulkanalia::prelude::v1_0::*;

// Vulkan 1.1
use vulkanalia::prelude::v1_1::*;

// Vulkan 1.2
use vulkanalia::prelude::v1_2::*;
```

## Validation layers

As mentioned earlier, Vulkan is designed for high performance and low driver overhead. Therefore it will include very limited error checking and debugging capabilities by default. The driver will often crash instead of returning an error code if you do something wrong, or worse, it will appear to work on your graphics card and completely fail on others.

Vulkan allows you to enable extensive checks through a feature known as *validation layers*. Validation layers are pieces of code that can be inserted between the API and the graphics driver to do things like running extra checks on function parameters and tracking memory management problems. The nice thing is that you can enable them during development and then completely disable them when releasing your application for zero overhead. Anyone can write their own validation layers, but the Vulkan SDK by LunarG provides a standard set of validation layers that we'll be using in this tutorial. You also need to register a callback function to receive debug messages from the layers.

Because Vulkan is so explicit about every operation and the validation layers are so extensive, it can actually be a lot easier to find out why your screen is black compared to OpenGL and Direct3D!
