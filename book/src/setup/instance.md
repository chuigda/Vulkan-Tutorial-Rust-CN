# Vulkan 实例

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/instance.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/01_instance_creation.rs)

你首先要做的事情就是通过创建一个 *实例* 来初始化 Vulkan 库。实例是你的应用程序和 Vulkan 库之间的连接，创建它涉及到向驱动程序指定一些关于你的应用程序的细节。首先，添加下面的导入：

```rust,noplaypen
use anyhow::{anyhow, Result};
use log::*;
use vulkanalia::loader::{LibloadingLoader, LIBRARY};
use vulkanalia::window as vk_window;
use vulkanalia::prelude::v1_0::*;
```

这里我们引入 [`anyhow!`](https://docs.rs/anyhow/latest/anyhow/macro.anyhow.html) 宏，这个宏可以让我们轻松地构造 `anyhow` 错误的实例。然后，我们引入 `log::*`，这样我们就可以使用 `log` crate 中的日志宏。接下来，我们引入 `LibloadingLoader`，它是 `vulkanalia` 提供的 `libloading` 集成，我们会用它来从 Vulkan 共享库中加载最初的 Vulkan 命令。你操作系统上 Vulkan 共享库（例如 Windows 上的 `vulkan-1.dll`）将会被导入为 `LIBRARY`。

接着我们将 `vulkanalia` 对窗口的集成导入为 `vk_window`，我们将在本章中使用它来枚举渲染到窗口所需的全局 Vulkan 扩展。在将来的章节中，我们还将使用 `vk_window` 来将 Vulkan 实例与 `winit` 窗口链接起来。

最后我们从 `vulkanalia` 引入 Vulkan 1.0 的 `prelude`  模块，它将为本章和将来的章节提供我们需要的所有其他 Vulkan 相关的导入。

要创建一个实例，我们就要用我们的应用程序的一些信息来填充一个 `vk::ApplicationInfo` 结构体。这些数据从技术上来说是可选的，但它可以给驱动程序提供一些有用的信息，以便优化我们的特定应用程序（例如我们的应用程序使用了某个众所周知的图形引擎，这个引擎具有某些特定的行为）。我们将在函数 `create_instance` 中创建 `vk::ApplicationInfo` 结构体，这个函数接受我们的窗口和一个 Vulkan 入口点（entry point）（我们将在后面创建）并返回一个 Vulkan 实例：

```rust,noplaypen
unsafe fn create_instance(window: &Window, entry: &Entry) -> Result<Instance> {
    let application_info = vk::ApplicationInfo::builder()
        .application_name(b"Vulkan Tutorial\0")
        .application_version(vk::make_version(1, 0, 0))
        .engine_name(b"No Engine\0")
        .engine_version(vk::make_version(1, 0, 0))
        .api_version(vk::make_version(1, 0, 0));
}
```

在 Vulkan 中，许多信息都是通过结构体而非函数参数传递的，所以我们再需要填充一个结构体，来提供创建一个实例所需的信息。下一个结构体是不可选的，它会告诉 Vulkan 驱动程序我们想要使用哪些全局扩展和校验层。这里的“全局”意味着这些扩展和校验层适用于整个程序，而不是特定的设备。“全局”和“设备”的概念将在接下来的几章中逐渐变得清晰。首先我们需要使用 `vulkanalia` 的窗口集成 `vk_window` 来枚举所需的全局扩展，并将它们转换为以空字符结尾的 C 字符串（null-terminated C strings）（`*const c_char`）：

```rust,noplaypen
let extensions = vk_window::get_required_instance_extensions(window)
    .iter()
    .map(|e| e.as_ptr())
    .collect::<Vec<_>>();
```

在有了所需的全局扩展列表之后，我们就可以使用传入此函数的 Vulkan 入口点来创建 Vulkan 实例并将其返回了：

```rust,noplaypen
let info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_extension_names(&extensions);

Ok(entry.create_instance(&info, None)?)
```

如你所见，Vulkan 中的对象创建函数参数的一般模式是：

* 包含创建信息的结构体的引用
* 可选的自定义分配器回调的引用，本教程中始终为 `None`

现在我们有了一个可以通过入口点创建 Vulkan 实例的函数，接下来我们需要创建一个 Vulkan 入口点。这个入口点将加载用于查询实例支持和创建实例的 Vulkan 命令。但在此之前，让我们向我们的 `App` 结构体添加一些字段来存储我们将要创建的 Vulkan 入口点和实例：

```rust,noplaypen
struct App {
    entry: Entry,
    instance: Instance,
}
```

接着，像这样更新 `App::create` 方法，以填充 `App` 中的这些字段：

```rust,noplaypen
unsafe fn create(window: &Window) -> Result<Self> {
    let loader = LibloadingLoader::new(LIBRARY)?;
    let entry = Entry::new(loader).map_err(|b| anyhow!("{}", b))?;
    let instance = create_instance(window, &entry)?;
    Ok(Self { entry, instance })
}
```

这里我们首先创建了一个 Vulkan 函数加载器，用来从 Vulkan 共享库中加载最初的 Vulkan 命令，接着我们使用这个函数加载器创建 Vulkan 入口点，这个入口点将会加载我们需要的所有 Vulkan 命令。最后，我们用 Vulkan 入口点调用 `create_instance` 函数来创建 Vulkan 实例。

## 清理工作

只有当程序将要退出时，`Instance` 实例才应该被销毁。可以在 `App::destroy` 方法中使用 `destroy_instance` 销毁实例：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.instance.destroy_instance(None);
}
```

和创建对象所用的 Vulkan 命令异样，用于销毁对象的 Vulkan 命令也接受一个可选的、指向自定义分配器回调的引用。所以和之前一样，我们传入 `None` 来使用默认的分配器行为。

## 不合规的 Vulkan 实现

不幸的是，并非每个平台都有一个完全符合 Vulkan 规范的 Vulkan API 的实现。在这样的平台上，可能会有一些标准的 Vulkan 特性是不可用的，或者 Vulkan 应用程序的实际行为与 Vulkan 规范有很大的不同。

在 Vulkan SDK 的 1.3.216 版本之后，使用不合规 Vulkan 实现的应用程序必须启用一些额外的 Vulkan 扩展。这些兼容性扩展的主要目的是强制开发人员承认他们的应用程序正在使用不合规的 Vulkan 实现，并且他们不期望一切都按 Vulkan 规范进行。

本教程会使用这些兼容性 Vulkan 扩展，这样你的程序就可以在缺少完全符合 Vulkan 实现的平台上运行了。

然而，你可能会问：“为什么我们要这么做？我们真的需要在一个入门级的 Vulkan 教程中考虑对小众平台的支持吗？”而事实证明，不那么小众的 macOS 就是那些缺少完全符合 Vulkan 实现的平台之一。

As was mentioned in the introduction, Apple has their own low-level graphics API, [Metal](https://en.wikipedia.org/wiki/Metal_(API)). The Vulkan implementation that is provided as part of the Vulkan SDK for macOS ([MoltenVK](https://moltengl.com/)) is a layer that sits in-between your application and Metal and translates the Vulkan API calls your application makes into Metal API calls. Because MoltenVK is [not fully conformant with the Vulkan specification](https://www.lunarg.com/wp-content/uploads/2022/05/The-State-of-Vulkan-on-Apple-15APR2022.pdf), you will need to enable the compatibility Vulkan extensions we've been talking about to support macOS. 

As an aside, while MoltenVK is not fully-conformant, you shouldn't encounter any issues caused by deviations from the Vulkan specification while following this tutorial on macOS.

## Enabling compatibility extensions

> **Note:** Even if you are not following this tutorial on a macOS, some of the code added in this section is referenced in the remainder of this tutorial so you can't just skip it!

We'll want to check if the version of Vulkan we are using is equal to or greather than the version of Vulkan that introduced the compatibility extension requirement. With this goal in mind, we'll first add an additional import:

```rust,noplaypen
use vulkanalia::Version;
```

With this new import in place, we'll define a constant for the minimum version:

```rust,noplaypen
const PORTABILITY_MACOS_VERSION: Version = Version::new(1, 3, 216);
```

Replace the extension enumeration and instance creation code with the following:

```rust,noplaypen
let mut extensions = vk_window::get_required_instance_extensions(window)
    .iter()
    .map(|e| e.as_ptr())
    .collect::<Vec<_>>();

// Required by Vulkan SDK on macOS since 1.3.216.
let flags = if 
    cfg!(target_os = "macos") && 
    entry.version()? >= PORTABILITY_MACOS_VERSION
{
    info!("Enabling extensions for macOS portability.");
    extensions.push(vk::KHR_GET_PHYSICAL_DEVICE_PROPERTIES2_EXTENSION.name.as_ptr());
    extensions.push(vk::KHR_PORTABILITY_ENUMERATION_EXTENSION.name.as_ptr());
    vk::InstanceCreateFlags::ENUMERATE_PORTABILITY_KHR
} else {
    vk::InstanceCreateFlags::empty()
};

let info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_extension_names(&extensions)
    .flags(flags);
```

This code enables `KHR_PORTABILITY_ENUMERATION_EXTENSION` if your application is being compiled for a platform that lacks a non-conformant Vulkan implementation (just checking for macOS here) and the Vulkan version meets or exceeds the minimum version we just defined.

This code also enables `KHR_GET_PHYSICAL_DEVICE_PROPERTIES2_EXTENSION` under the same conditions. This extension is needed to enable the `KHR_PORTABILITY_SUBSET_EXTENSION` device extension which will be added later in the tutorial when we set up a logical device.

## `Instance` vs `vk::Instance`

When we call our `^create_instance` function, what we get back is not a raw Vulkan instance as would be returned by the Vulkan command `vkCreateInstance` (`vk::Instance`). Instead what we got back is a custom type defined by `vulkanalia` which combines both a raw Vulkan instance and the commands loaded for that specific instance.

This is the `Instance` type we have been using (imported from the `vulkanalia` prelude) which should not be confused with the `vk::Instance` type which represents a raw Vulkan instance. In future chapters we will also use the `Device` type which, like `Instance`, is a pairing of a raw Vulkan device (`vk::Device`) and the commands loaded for that specific device. Fortunately we will not be using `vk::Instance` or `vk::Device` directly in this tutorial so you don't need to worry about getting them mixed up.

Because an `Instance` contains both a Vulkan instance and the associated commands, the command wrappers implemented for an `Instance` are able to provide the Vulkan instance when it is required by the underlying Vulkan command.

If you look at the documentation for the `vkDestroyInstance` command, you will see that it takes two parameters: the instance to destroy and the optional custom allocator callbacks. However, `destroy_instance` only takes the optional custom allocator callbacks because it is able to provide the raw Vulkan handle as the first parameter itself as described above.

Before continuing with the more complex steps after instance creation, it's time to evaluate our debugging options by checking out validation layers.
