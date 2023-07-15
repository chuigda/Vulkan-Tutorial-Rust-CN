# 校验层

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/validation_layers.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**代码：**[main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/02_validation_layers.rs)

Vulkan API 的设计秉持了尽可能降低驱动开销的理念，带来的影响就是 API 默认只提供极少的错误检查。即便是像把枚举设置成了一个非法值这样简单的错误也不会被显式处理，而是会导致程序崩溃或是未定义行为。由于 Vulkan 要求你明确你所做的事，你很容易就会犯下许多小错误，例如在使用一个新的 GPU 特性时忘记在创建逻辑设备时请求这个特性。

然而，这并不意味着 API 上就没法进行错误检查。Vulkan 引入了一个叫做校验层（Validation Layer）的优雅的系统。校验层是可选的，它们能在 Vulkan 函数调用时插入钩子，用来执行额外的操作。一些通常的操作包括：

* 对比规范检查参数值，以检测是否有误用
* 追踪对象的创建和销毁，找出资源泄漏
* 通过追踪发起调用的线程，检查线程安全性
* 在标准输出中打印含有所有调用极其参数的日志
* 追踪 Vulkan 调用，用于分析性能（profiling）与重放（replay）

诊断校验层中一个函数的实现看起来像这样（C 语言）：

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

你可以随意堆叠校验层来引入你感兴趣的调试功能。你只需要为 Debug 构建启用校验层，然后在 Release 构建把它们禁用。这样就能使这两个构建获得最大收益。

Vulkan 并不内置任何校验层，但是 LunarG Vulkan SDK 提供了一系列用以检查通用错误的校验层。它们是完全[开源](https://github.com/KhronosGroup/Vulkan-ValidationLayers)的，所以你可以找到它们能检查的错误类型，并且可以参与贡献。你的应用可能会因为无意中用到了未定义行为而在不同驱动上发生错误。避免这种事发生的最好方式就是使用校验层。

校验层只能在安装到系统中之后使用。比如 LunarG 校验层只能在安装了 Vulkan SDK 的电脑上使用。

之前，在 Vulkan 中有两种不同类型的校验层：实例（instance）特定层与设备（device）特定层。实例特定层只会检查与全局 Vulkan 对象（例如实例）有关的调用，而设备特定层只会检查与一个特定 GPU 有关的调用。设备特定层现在已经被弃用了，意味着实例特定层能应用在所有 Vulkan 调用上。规范文档依旧建议你为兼容性启用设备特定层，而在某些实现中它是必须的。我们将简单地在逻辑设备级别指定与实例相同的层，稍后我们会看到。

在开始之前，我们需要为本章节添加一些新的引入：

```rust,noplaypen
use std::collections::HashSet;
use std::ffi::CStr;
use std::os::raw::c_void;

use vulkanalia::vk::ExtDebugUtilsExtension;
```

`HashSet` 会被用在存储与查询支持的校验层，`vk::ExtDebugUtilsExtension` 提供管理调试功能的指令封装。其它引入会被用于记录校验层传来的信息。

## 使用校验层

在这个章节中，我们会学到如何启用 Vulkan SDK 提供的标准诊断层。与扩展一样，校验层需要指定它们的名称以启用。在 SDK 中，所有有用的标准校验都被打包于 `VK_LAYER_KHRONOS_validation` 校验层中。

我们先给我们程序增加两个配置变量，一个用来指定需要启用的校验层，一个用来指定是否启用校验层。我决定根据程序是否使用 Debug 模式编译来选择是否启用校验层。

```rust,noplaypen
const VALIDATION_ENABLED: bool =
    cfg!(debug_assertions);

const VALIDATION_LAYER: vk::ExtensionName =
    vk::ExtensionName::from_bytes(b"VK_LAYER_KHRONOS_validation");
```

我们给我们的 `create_instance` 函数加一些新的代码，用来收集所有支持的实例层，并将其存储在一个 `HashSet` 中，检查这些校验层是否可用，并且创建一个包含校验层名称的列表。以下代码应该放在构建 `vk::ApplicationInfo` 结构体的正下方：

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

然后，你需要在 `vk::InstanceCreateInfo` 中指定请求的校验层。我们需要调用 `enabled_layer_names` 构建方法：

```rust,noplaypen
let info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_layer_names(&layers)
    .enabled_extension_names(&extensions)
    .flags(flags);
```

现在，在 Debug 模式下执行程序，并且确保没有跳出 `Validation layer requested but not supported.` 这条错误信息。如果看到报错信息，那你需要看一下 FAQ。如果你通过了这条检查，那 `create_instance` 应当不会返回错误代码 `vk::ErrorCode::LAYER_NOT_PRESENT`，但是你依旧应该运行程序以确保没有这个错误代码。

## 消息回调

校验层默认会打印调试消息至标准输出，但我们也可以通过提供显式回调的方式自己处理这些消息。这样我们可以自己决定处理哪些类型的消息，因为并非所有消息是（致命）错误消息。如果你不想现在做这些事，你可以跳到本章的最后一节。

为了在程序中配置一个处理消息与附带细节的回调，我们需要使用 `VK_EXT_debug_utils` 扩展配置一个带回调的调试信使（debug messenger）。

我们会对 `create_instance` 函数添加更多代码。这次我们需要将 `extensions` 列表改为可变的，然后在校验层启用时将调试实用工具扩展 （debug utilities extension）加入这个列表：

```rust,noplaypen
let mut extensions = vk_window::get_required_instance_extensions(window)
    .iter()
    .map(|e| e.as_ptr())
    .collect::<Vec<_>>();

if VALIDATION_ENABLED {
    extensions.push(vk::EXT_DEBUG_UTILS_EXTENSION.name.as_ptr());
}
```

`vulkanalia` provides a collection of metadata for each Vulkan extension. In this case we just need the name of the extension to load, so we add the value of the `name` field of the `vk::EXT_DEBUG_UTILS_EXTENSION` struct constant to our list of desired extension names.

Run the program to make sure you don't receive a `vk::ErrorCode::EXTENSION_NOT_PRESENT` error code. We don't really need to check for the existence of this extension, because it should be implied by the availability of the validation layers.

Now let's see what a debug callback function looks like. Add a new `extern "system"` function called `debug_callback` that matches the `vk::PFN_vkDebugUtilsMessengerCallbackEXT` prototype. The `extern "system"` is necessary to allow Vulkan to call our Rust function.

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

The first parameter specifies the severity of the message, which is one of the following flags:

* `vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE` &ndash; Diagnostic message
* `vk::DebugUtilsMessageSeverityFlagsEXT::INFO` &ndash; Informational message like the creation of a resource
* `vk::DebugUtilsMessageSeverityFlagsEXT::WARNING` &ndash; Message about behavior that is not necessarily an error, but very likely a bug in your application
* `vk::DebugUtilsMessageSeverityFlagsEXT::ERROR` &ndash; Message about behavior that is invalid and may cause crashes

The values of this enumeration are set up in such a way that you can use a comparison operation to check if a message is equal or worse compared to some level of severity which we use here to decide on which `log` macro is appropriate to use when logging the message.

The `type_` parameter can have the following values:

* `vk::DebugUtilsMessageTypeFlagsEXT::GENERAL` &ndash; Some event has happened that is unrelated to the specification or performance
* `vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION` &ndash; Something has happened that violates the specification or indicates a possible mistake
* `vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE` &ndash; Potential non-optimal use of Vulkan

The `data` parameter refers to a `vk::DebugUtilsMessengerCallbackDataEXT` struct containing the details of the message itself, with the most important members being:

* `message` &ndash; The debug message as a null-terminated string (`*const c_char`)
* `objects` &ndash; Array of Vulkan object handles related to the message
* `object_count` &ndash; Number of objects in array

Finally, the last parameter, here ignored as `_`, contains a pointer that was specified during the setup of the callback and allows you to pass your own data to it.

The callback returns a (Vulkan) boolean that indicates if the Vulkan call that triggered the validation layer message should be aborted. If the callback returns true, then the call is aborted with the `vk::ErrorCode::VALIDATION_FAILED_EXT` error code. This is normally only used to test the validation layers themselves, so you should always return `vk::FALSE`.

All that remains now is telling Vulkan about the callback function. Perhaps somewhat surprisingly, even the debug callback in Vulkan is managed with a handle that needs to be explicitly created and destroyed. Such a callback is part of a debug messenger and you can have as many of them as you want. Add a field to the `AppData` struct:

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

> **注：**在 Vulkan 旗标（例如上面的例子中的 `vk::DebugUtilsMessageTypeFlagsEXT::all()`）上调用 `all` 静态方法会返回一系列 `vulkanalia` 可识别的旗标。这些旗标会在返回的对应二进制位上被设为 1。
> **Note:** Calling the `all` static method on a set of Vulkan flags (e.g., `vk::DebugUtilsMessageTypeFlagsEXT::all()` as in the above example) will return a set of flags with all of the bits set for the flags known by `vulkanalia`. This introduces the possibility that if the application uses an implementation of Vulkan older than the latest version of Vulkan that `vulkanalia` supports, this set of flags could include flags that aren't known by the Vulkan implementation in use by the application. This shouldn't cause any issues with the functionality of the application, but you might see some validation errors. If you encounter warnings about unknown flags because of these debug flags, you can avoid them by upgrading your Vulkan SDK to the latest version (or directly specifying the supported flags).

We have first extracted our Vulkan instance out of the return expression so we can use it to add our debug callback.

Then we construct a `vk::DebugUtilsMessengerCreateInfoEXT` struct which provides information about our debug callback and how it will be called.

The `message_severity` field allows you to specify all the types of severities you would like your callback to be called for. I've requested that messages of all severity be included. This would normally produce a lot of verbose general debug info but we can filter that out using a log level when we are not interested in it.

Similarly the `message_type` field lets you filter which types of messages your callback is notified about. I've simply enabled all types here. You can always disable some if they're not useful to you.

Finally, the `user_callback` field specifies the callback function. You can optionally pass a mutable reference to the `user_data` field which will be passed along to the callback function via the final parameter. You could use this to pass a pointer to the `AppData` struct, for example.

Lastly we call `create_debug_utils_messenger_ext` to register our debug callback with the Vulkan instance.

Since our `^create_instance` function takes an `AppData` reference now, we'll also need to update `App` and `App::create`:

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

The `vk::DebugUtilsMessengerEXT` object we created needs to cleaned up before our app exits. We'll do this in `App::destroy` before we destroy the instance:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    if VALIDATION_ENABLED {
        self.instance.destroy_debug_utils_messenger_ext(self.data.messenger, None);
    }

    self.instance.destroy_instance(None);
}
```

## 创建与销毁调试实例

Although we've now added debugging with validation layers to the program we're not covering everything quite yet. The `create_debug_utils_messenger_ext` call requires a valid instance to have been created and `destroy_debug_utils_messenger_ext` must be called before the instance is destroyed. This currently leaves us unable to debug any issues in the `create_instance` and `destroy_instance` calls.

However, if you closely read the [extension documentation](https://github.com/KhronosGroup/Vulkan-Docs/blob/77d9f42e075e6a483a37351c14c5e9e3122f9113/appendices/VK_EXT_debug_utils.txt#L84-L91), you'll see that there is a way to create a separate debug utils messenger specifically for those two function calls. It requires you to simply pass a pointer to a `vk::DebugUtilsMessengerCreateInfoEXT` struct in the `next` extension field of `vk::InstanceCreateInfo`. Before we do this, let's first discuss how extending structs works in Vulkan.

The `s_type` field that is present on many Vulkan structs was briefly mentioned in the [Builders section](../overview.html#builders) of the Overview chapter. It was said that this field must be set to the `vk::StructureType` variant indicating the type of the struct (e.g., `vk::StructureType::APPLICATION_INFO` for a `vk::ApplicationInfo` struct).

You may have wondered what the purpose of this field is: doesn't Vulkan already know the type of structs passed to its commands? The purpose of this field is wrapped up with the purpose of the `next` field that always accompanies the `s_type` field in Vulkan structs: the ability to *extend* a Vulkan struct with other Vulkan structs.

The `next` field in a Vulkan struct may be used to specify a [structure pointer chain](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#fundamentals-validusage-pNext). `next` can be either be null or a pointer to a Vulkan struct that is permitted by Vulkan to extend the struct. Each struct in this chain of structs is used to provide additional information to the Vulkan command the root structure is passed to. This feature of Vulkan allows for extending the functionality of Vulkan commands without breaking backwards compabilitity.

When you pass such a chain of structs to a Vulkan command, it must iterate through the structs to collect all of the information from the structs. Because of this, Vulkan can't know the type of each structure in the chain, hence the need for the `s_type` field.

The builders provided by `vulkanalia` allow for easily building these pointer chains in a type-safe manner. For example, take a look at the `vk::InstanceCreateInfoBuilder` builder, specifically the `push_next` method. This method allows adding any Vulkan struct for which the `vk::ExtendsInstanceCreateInfo` trait is implemented for to the pointer chain for a `vk::InstanceCreateInfo`.

One such struct is `vk::DebugUtilsMessengerCreateInfoEXT`, which we will now use to extend our `vk::InstanceCreateInfo` struct to set up our debug callback. To do this we'll continue to modify our `^create_instance` function. This time we'll make the `info` struct mutable so we can modify its pointer chain before moving the `debug_info` struct, now also mutable, below it so we can push it onto `info`'s pointer chain:

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

`debug_info` needs to be defined outside of the conditional since it needs to live until we are done calling `create_instance`. Fortunately we can rely on the Rust compiler to protect us from pushing a struct that doesn't live long enough onto a pointer chain due to the lifetimes defined for the `vulkanalia` builders.

Now we should be able to run our program and see logs from our debug callback, but first we'll need to set the `RUST_LOG` environment variable so that `pretty_env_logger` will enable the log levels we are interested in. Initially set the log level to `debug` so we can be sure it is working, here is an example on Windows (PowerShell):

![](../images/validation_layer_test.png)

If everything is working you shouldn't see any warning or error messages. Going forward you will probably want to increase the minimum log level to `info` using `RUST_LOG` to reduce the verbosity of the logs unless you are trying to debug an error.

## 配置

There are a lot more settings for the behavior of validation layers than just the flags specified in the `vk::DebugUtilsMessengerCreateInfoEXT` struct. Browse to the Vulkan SDK and go to the `Config` directory. There you will find a `vk_layer_settings.txt` file that explains how to configure the layers.

To configure the layer settings for your own application, copy the file to the working directory of your project's executable and follow the instructions to set the desired behavior. However, for the remainder of this tutorial I'll assume that you're using the default settings.

Throughout this tutorial I'll be making a couple of intentional mistakes to show you how helpful the validation layers are with catching them and to teach you how important it is to know exactly what you're doing with Vulkan. Now it's time to look at Vulkan devices in the system.
