# 校验层

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/validation_layers.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:**[main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/02_validation_layers.rs)

Vulkan API 的设计秉持了尽可能降低驱动开销的理念，带来的影响就是 API 默认只提供极少的错误检查。即便是像把枚举设置成了一个非法值这样简单的错误也不会被显式处理，而是会导致程序崩溃或是未定义行为。由于 Vulkan 要求你明确你所做的事，你很容易就会犯下许多小错误，例如在使用一个新的 GPU 特性时忘记在创建逻辑设备时请求这个特性。

然而，这并不意味着 Vulkan API 就没法进行错误检查。Vulkan 引入了一个优雅的系统，叫做校验层（Validation Layer）。校验层是可选的，它们能在你调用 Vulkan 函数时插入钩子，执行额外的操作。一些通常的操作包括：

* 对比规范检查参数值，以检测是否有误用
* 追踪对象的创建和销毁，找出资源泄漏
* 通过追踪发起调用的线程，检查线程安全性
* 在标准输出中打印含有所有调用及其参数的日志
* 追踪 Vulkan 调用，用于分析性能（profiling）与重放（replay）

诊断校验层中一个函数的实现看起来就像这样（C 语言）：

```c
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance
) {
    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

你可以随意堆叠校验层来引入你感兴趣的调试功能。你只需为 Debug 构建启用校验层，而在 Release 构建禁用它们，就能使这两个构建获得最大收益。

Vulkan 并不内置任何校验层，但是 LunarG Vulkan SDK 提供了一系列校验层，用以检查常见的错误。它们是完全[开源](https://github.com/KhronosGroup/Vulkan-ValidationLayers)的，所以你可以找到它们能检查的错误类型，并且可以参与贡献。你的应用可能会因为无意中依赖于未定义行为而在不同的驱动程序上遇到错误，而要避免这种事，最好的方式就是使用校验层。

校验层只能在安装到系统中之后使用。比如 LunarG 校验层只能在安装了 Vulkan SDK 的电脑上使用。

之前，在 Vulkan 中有两种不同类型的校验层：实例（instance）特定的校验层与设备（device）特定的校验特定层。实例特定的校验层会检查与全局 Vulkan 对象 —— 例如 Vulkan 实例 —— 相关的调用，而设备特定的校验层只会检查与某个特定的 GPU 相关的调用。设备特定的校验层现在已经被弃用了，这也就意味着实例特定的校验层会对所有的 Vulkan 调用生效。规范文档依旧建议你出于兼容性考虑启用设备特定的校验层，而在某些实现中它是必须的。我们将简单地在逻辑设备级别指定与实例相同的校验层，稍后我们会看到。

在开始之前，我们需要为本章节添加一些新的引入：

```rust,noplaypen
use std::collections::HashSet;
use std::ffi::CStr;
use std::os::raw::c_void;

use vulkanalia::vk::ExtDebugUtilsExtension;
```

`HashSet` 会被用在存储与查询支持的校验层，`vk::ExtDebugUtilsExtension` 提供管理调试功能的指令封装。其它引入会被用于记录校验层传来的信息。

## 使用校验层

在这个章节中，我们会学到如何启用 Vulkan SDK 提供的标准校验层。和扩展一样，校验层需要指定它们的名称以启用。在 SDK 中，所有有用的标准校验都被打包于 `VK_LAYER_KHRONOS_validation` 校验层中。

我们先给我们程序增加两个配置变量，一个用来指定需要启用的校验层，一个用来指定是否启用校验层。我决定根据程序是否使用 Debug 模式编译来选择是否启用校验层。

```rust,noplaypen
const VALIDATION_ENABLED: bool =
    cfg!(debug_assertions);

const VALIDATION_LAYER: vk::ExtensionName =
    vk::ExtensionName::from_bytes(b"VK_LAYER_KHRONOS_validation");
```

我们给 `create_instance` 函数加一些新的代码，用来收集所有支持的实例特定校验层并将其存储在一个 `HashSet` 中，然后使用这个 `HashSet` 检查我们需要的校验层是否可用，并创建一个包含校验层名称的列表。这些代码应该放在构建 `vk::ApplicationInfo` 结构体的正下方：

```rust,noplaypen
let available_layers = entry
    .enumerate_instance_layer_properties()?
    .iter()
    .map(|l| l.layer_name)
    .collect::<HashSet<_>>();

if VALIDATION_ENABLED && !available_layers.contains(&VALIDATION_LAYER) {
    return Err(anyhow!("Validation layer requested but not supported."));
}

let layers = if VALIDATION_ENABLED {
    vec![VALIDATION_LAYER.as_ptr()]
} else {
    Vec::new()
};
```

然后，你需要调用 `enabled_layer_names`，在 `vk::InstanceCreateInfo` 中指定需要启用的校验层：

```rust,noplaypen
let info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_layer_names(&layers)
    .enabled_extension_names(&extensions)
    .flags(flags);
```

现在，在 Debug 模式下执行程序，并且确保没有跳出 `Validation layer requested but not supported.` 这条错误信息。如果看到报错信息，那你需要看一下 FAQ。如果一切顺利，那么 `create_instance` 应该永远都不会返回错误代码 `vk::ErrorCode::LAYER_NOT_PRESENT`，不过你还是应该运行程序以确保万无一失。

## 消息回调

默认情况下，校验层会将调试消息打印至标准输出，但我们也可以提供显式回调自己处理这些消息。这样我们可以自主决定处理哪些类型的消息，因为并非所有消息都是（致命）错误消息。如果你不想现在做这些事，你可以跳到本章的最后一节。

为了在程序中配置一个处理消息和消息细节的回调，我们需要使用 `VK_EXT_debug_utils` 扩展配置一个带回调的调试信使（debug messenger）。

我们会往 `create_instance` 函数中函数添加更多代码。这次我们需要将 `extensions` 列表改为可变的，然后在校验层启用时将调试实用工具扩展（debug utilities extension）加入这个列表：

```rust,noplaypen
let mut extensions = vk_window::get_required_instance_extensions(window)
    .iter()
    .map(|e| e.as_ptr())
    .collect::<Vec<_>>();

if VALIDATION_ENABLED {
    extensions.push(vk::EXT_DEBUG_UTILS_EXTENSION.name.as_ptr());
}
```

`vulkanalia` 为每个 Vulkan 扩展提供了一系列元数据。在这个例子中，我们只需要加载扩展的名称，所以我们将 `vk::EXT_DEBUG_UTILS_EXTENSION` 结构体常量的 `name` 字段的值添加到我们的扩展名称列表中。

运行程序，确保你没有收到 `vk::ErrorCode::EXTENSION_NOT_PRESENT` 错误代码。事实上，我们并不需要检查这个扩展是否存在，因为只要校验层可用，这个扩展应该就是可用的。

现在让我们看看一个调试回调函数是什么样的。添加一个新的 `extern "system"` 函数，叫做 `debug_callback`，它的签名与 `vk::PFN_vkDebugUtilsMessengerCallbackEXT` 原型相匹配。`extern "system"` 是必须的，这样 Vulkan 才能正确调用我们的 Rust 函数。

```rust,noplaypen
extern "system" fn debug_callback(
    severity: vk::DebugUtilsMessageSeverityFlagsEXT,
    type_: vk::DebugUtilsMessageTypeFlagsEXT,
    data: *const vk::DebugUtilsMessengerCallbackDataEXT,
    _: *mut c_void,
) -> vk::Bool32 {
    let data = unsafe { *data };
    let message = unsafe { CStr::from_ptr(data.message) }.to_string_lossy();

    if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::ERROR {
        error!("({:?}) {}", type_, message);
    } else if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::WARNING {
        warn!("({:?}) {}", type_, message);
    } else if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::INFO {
        debug!("({:?}) {}", type_, message);
    } else {
        trace!("({:?}) {}", type_, message);
    }

    vk::FALSE
}
```

第一个参数表示消息的严重程度，它可以有以下取值：

* `vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE` &ndash; 诊断信息
* `vk::DebugUtilsMessageSeverityFlagsEXT::INFO` &ndash; 提示消息，例如资源的创建
* `vk::DebugUtilsMessageSeverityFlagsEXT::WARNING` &ndash; 行为不一定是错误，但很可能意味着你的程序有 bug
* `vk::DebugUtilsMessageSeverityFlagsEXT::ERROR` &ndash; 行为无效，可能会导致崩溃

枚举值被设置为递增的，这样就可以用比较运算符来检查一条消息是否比某个严重程度更严重。我们根据这一点来决定在记录消息时使用哪个 `log` 宏。

`_type` 参数可以有以下取值：

* `vk::DebugUtilsMessageTypeFlagsEXT::GENERAL` &ndash; 与规范或性能无关的事件
* `vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION` &ndash; 违反规范或可能是错误的事件
* `vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE` &ndash; 这个用法可能不是 Vulkan 的最佳实践

`data` 参数指向一个 `vk::DebugUtilsMessengerCallbackDataEXT` 结构体，它包含了消息本身的细节，其中最重要的成员是：

* `message` &ndash; 调试信息，以空字符结尾的 C 字符串（`*const c_char`）
* `objects` &ndash; 与消息相关的 Vulkan 对象句柄数组
* `object_count` &ndash; 数组中对象的数量

最后，最后一个参数，这里被忽略为 `_`，包含了一个指针，它在回调函数设置时被指定，允许你将自己的数据传递给它。

回调函数返回一个布尔值，它表示触发校验层消息的 Vulkan 调用是否应该被中止。如果回调函数返回 `true`，那么这个调用就会被中止，并且返回 `vk::ErrorCode::VALIDATION_FAILED_EXT` 错误代码。这通常只用于测试校验层本身，所以你应该总是返回 `vk::FALSE`。

现在我们需要告诉 Vulkan 关于回调函数的事情。也许有点出乎意料，在 Vulkan 中，即使是调试回调函数也需要显式创建和销毁。这样的回调函数是调试信使（debug messenger）的一部分，你可以有任意多个这样的回调函数。在 `AppData` 结构体中添加一个字段：

```rust,noplaypen
struct AppData {
    messenger: vk::DebugUtilsMessengerEXT,
}
```

修改 `create_instance` 函数的签名与结尾，让它变得像这样：

```rust,noplaypen
unsafe fn create_instance(
    window: &Window,
    entry: &Entry,
    data: &mut AppData
) -> Result<Instance> {
    // ...

    let instance = entry.create_instance(&info, None)?;

    if VALIDATION_ENABLED {
        let debug_info = vk::DebugUtilsMessengerCreateInfoEXT::builder()
            .message_severity(vk::DebugUtilsMessageSeverityFlagsEXT::all())
            .message_type(vk::DebugUtilsMessageTypeFlagsEXT::all())
            .user_callback(Some(debug_callback));

        data.messenger = instance.create_debug_utils_messenger_ext(&debug_info, None)?;
    }

    Ok(instance)
}
```

<!-- TODO needs refinement -->
> **注：**在一组 Vulkan 标志（例如上面的例子中的 `vk::DebugUtilsMessageTypeFlagsEXT::all()`）上调用 `all` 静态方法会返回一系列 `vulkanalia` 可识别的标志。这会导致一个问题，如果应用程序使用的 Vulkan 实现比 `vulkanalia` 支持的最新版本的 Vulkan 旧，那么这个标志集可能包含应用程序使用的 Vulkan 实现不认识的标志。这不会影响应用程序的功能，但是你可能会看到一些验证错误。如果你因为这些调试标志遇到了未知标志的警告，你可以通过升级你的 Vulkan SDK 到最新版本（或者直接指定支持的标志）来避免这些警告。

首先，我们从返回表达式中提取出 Vulkan 实例，这样我们就可以用它来添加我们的调试回调函数。

接着，我们构造一个 `vk::DebugUtilsMessengerCreateInfoEXT` 结构体，它提供了和我们的调试回调函数以及如何调用这个函数有关的信息。

`message_severity` 字段允许你指定你想要你的回调函数感兴趣的所有严重程度类型。我请求所有严重程度的消息都被包含。这通常会产生大量的冗长的调试信息，但是我们可以在不感兴趣的时候通过日志级别来过滤掉这些信息。

类似地，`message_type` 字段允许你过滤你的回调函数感兴趣的消息类型。我在这里启用了所有类型。如果某些类型的消息对你没用，你可以禁用它们。

最后，`user_callback` 字段指定了回调函数。你可以选择传递一个可变引用给 `user_data` 字段，它会通过最后一个参数传递给回调函数。比如，你可以使用这个来传递一个指向 `AppData` 结构体的指针。

最后，我们调用 `create_debug_utils_messenger_ext` 来把我们的调试回调函数注册到 Vulkan 实例中。

因为 `create_instance` 函数接受一个 `AppData` 的引用，我们还需要更新 `App` 和 `App::create`：

```rust,noplaypen
struct App {
    entry: Entry,
    instance: Instance,
    data: AppData,
}

impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        let mut data = AppData::default();
        let instance = create_instance(window, &entry, &mut data)?;
        Ok(Self { entry, instance, data })
    }
}
```

在程序退出前，我们创建的 `vk::DebugUtilsMessengerEXT` 对象需要被清理。我们会在 `App::destroy`中，在销毁实例之前做这件事：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    if VALIDATION_ENABLED {
        self.instance.destroy_debug_utils_messenger_ext(self.data.messenger, None);
    }

    self.instance.destroy_instance(None);
}
```

## 创建与销毁调试实例

尽管我们已经通过校验层添加了调试功能，但活还没完全干完。调用 `create_debug_utils_messenger_ext` 需要一个有效的实例，而 `destroy_debug_utils_messenger_ext` 必须在实例被销毁前调用。这意味着我们现在还不能调试 `create_instance` 和 `destroy_instance` 调用中的任何问题。

不过，如果你仔细阅读过 [扩展文档](https://github.com/KhronosGroup/Vulkan-Docs/blob/77d9f42e075e6a483a37351c14c5e9e3122f9113/appendices/VK_EXT_debug_utils.txt#L84-L91)，你就会看到，还有一种方式可以为这两个函数调用创建一个单独的调试信使。只需在 `vk::InstanceCreateInfo` 的 `next` 扩展字段中传递一个指向 `vk::DebugUtilsMessengerCreateInfoEXT` 结构体的指针即可。在这么做之前，我们先来讨论一下在 Vulkan 中如何扩展结构体。

我们在概览章节的[生成器](../overview.html#builders)那一节提到过，许多 Vulkan 结构体中都有一个 `s_type` 字段。它必须设置成正确的 `vk::StructureType` 枚举变体，以指示结构体的类型（例如，`vk::ApplicationInfo` 结构体的 `s_type` 字段必须设置成 `vk::StructureType::APPLICATION_INFO`）。

你可能好奇过这个字段的目的是什么：Vulkan 不是已经知道传递给它的结构体的类型了吗？事实上，这个字段与 `next` 字段的目的紧密相关：它提供了扩展 Vulkan 结构体的能力。

Vulkan 结构体中的 `next` 字段可以用来指定一个[结构体指针链](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#fundamentals-validusage-pNext)。`next` 可以是空指针，也可以是一个指向 Vulkan 结构体的指针，Vulkan 可以利用这一点来扩展结构体。这个链中的每个结构体都可以为传递给 Vulkan 命令的根结构体提供额外的信息。Vulkan 的这个特性允许在不破坏向后兼容性的情况下扩展 Vulkan 命令的功能。

当你将这样的结构体链传递给 Vulkan 命令时，Vulkan 命令将会遍历结构体链以收集链中所有结构体的信息。因此，Vulkan 不知道链中每个结构体的类型，这就需要 `s_type` 字段。

`vulkanalia` 提供的生成器能够轻松地以类型安全的方式构建这样的结构体链。例如，看一下 `vk::InstanceCreateInfoBuilder` 生成器，特别是 `push_next` 方法。这个方法允许将任何实现了 `vk::ExtendsInstanceCreateInfo` 特性的 Vulkan 结构体添加到 `vk::InstanceCreateInfo` 的结构体链中。

`vk::DebugUtilsMessengerCreateInfoEXT` 便是这种结构体之一，我们会用它来扩展 `vk::InstanceCreateInfo` 结构，以设置我们的调试回调函数。为了做到这一点，继续修改 `create_instance` 函数，这一次我们会把 `info` 结构体变成可变的，这样我们就可以修改它的指针链。然后将 `debug_info` 结构体 —— 现在也是可变的 —— 放在 `info` 结构体的下面，这样我们就可以将它推到 `info` 的指针链上：

```rust,noplaypen
let mut info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_layer_names(&layers)
    .enabled_extension_names(&extensions)
    .flags(flags);

let mut debug_info = vk::DebugUtilsMessengerCreateInfoEXT::builder()
    .message_severity(vk::DebugUtilsMessageSeverityFlagsEXT::all())
    .message_type(vk::DebugUtilsMessageTypeFlagsEXT::all())
    .user_callback(Some(debug_callback));

if VALIDATION_ENABLED {
    info = info.push_next(&mut debug_info);
}
```

`debug_info` 需要在条件语句之外定义，因为它需要一直存活到我们调用完 `create_instance` 之后。幸运的是，我们可以依赖 Rust 编译器来保护我们：因为 `vulkanalia` 生成器定义的生命周期而，我们无法将一个活得不够长的结构体推到指针链上。

现在我们可以运行程序，观察调试回调函数打印的日志了。不过我们要先设置 `RUST_LOG` 环境变量，这样 `pretty_env_logger` 就会启用我们感兴趣的日志级别。我们先把日志级别设置为 `debug`，以确定这些东西工作正常。下面是在 Windows（PowerShell）上的一个例子：

![](../images/validation_layer_test.png)

如果一切正常，那你应该不会看到任何警告或者错误信息。接下来，你可能想要使用 `RUST_LOG=info` 来减少日志的冗长程度，除非你是在调试错误。

## 配置

除了 `vk::DebugUtilsMessengerCreateInfoEXT` 结构中的标志之外，还有更多针对校验层行为的配置项目。浏览 Vulkan SDK 所在的位置，进入 `Config` 目录。你会在这里找到一个 `vk_layer_settings.txt` 文件，它解释了如何配置校验层的行为。

要为你自己的应用程序配置校验层，你需要将文件复制到项目可执行文件的工作目录，并按照说明设置所需的行为。不过，在本教程的其余部分，我将假设你使用默认设置。

在本教程中，我会故意制造一些错误，以展示校验层是如何帮助你捕获这些错误的，并告诉你与 Vulkan 共事时清楚自己在做什么是多么重要。现在是时候看看系统中的 Vulkan 设备了。
