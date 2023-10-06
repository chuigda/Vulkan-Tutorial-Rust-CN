# 物理设备与队列族

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/physical_devices_and_queue_families.html>
>
> Commit Hash: ceb4a3fc6d8ca565af4f8679c4889bcad7941338

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/03_physical_device_selection.rs)

在通过 `Instance` 初始化 Vulkan 库之后，我们需要在系统中选择一个支持我们所需功能的图形处理器。事实上，我们可以选择任意多个图形处理器，并同时使用它们，不过在本教程中我们只会选择第一个满足我们需求的图形处理器。

我们会添加一个 `pick_physical_device` 函数，用来枚举并选择图形处理器，然后将图形处理器及其相关信息存储在 `AppData` 中。这个函数及其调用的函数会使用一个自定义的错误类型（`SuitabilityError`）来表示物理设备不满足应用程序的需求。这个错误类型会使用 `thiserror` crate 来自动实现错误类型需要的所有的样板代码。

```rust,noplaypen
use thiserror::Error;

impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        pick_physical_device(&instance, &mut data)?;
        Ok(Self { entry, instance, data })
    }
}

#[derive(Debug, Error)]
#[error("Missing {0}.")]
pub struct SuitabilityError(pub &'static str);

unsafe fn pick_physical_device(instance: &Instance, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

被选中的物理设备会被存储在我们刚添加到 `AppData` 结构体的 `vk::PhysicalDevice` 句柄中。当 `Instance` 被销毁时，这个对象也会被隐式销毁，所以我们不需要在 `App::destroy` 方法中做任何额外的工作。

```rust,noplaypen
struct AppData {
    // ...
    physical_device: vk::PhysicalDevice,
}
```

## 设备的适用性

我们需要一种方法来确定一个物理设备是否符合我们的需求。我们会创建一个用来检测设备适用性的函数，如果我们传给这个函数的物理设备不能完全支持我们需要的功能，那么这个函数会返回一个 `SuitabilityError` 错误：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    Ok(())
}
```

要评估一个物理设备是否满足我们的需求，我们可以从设备中查询一些详细信息设备的基本信息，例如名称、类型和支持的 Vulkan 版本都可以使用 `get_physical_device_properties` 查询：

```rust,noplaypen
let properties = instance
    .get_physical_device_properties(physical_device);
```

设备对可选特性，例如纹理压缩、64 位浮点类型和多视口渲染（在 VR 中很有用）的支持可以使用 `get_physical_device_features` 查询：

```rust,noplaypen
let features = instance
    .get_physical_device_features(physical_device);
```

我们会在讨论设备内存和队列族（见下一节）的时候再讨论更多可以查询的设备细节。

举个例子，假设我们的应用程序只能在支持几何着色器（geometry shader）的独立显卡上运行。那么 `check_physical_device` 函数可能如下所示：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    let properties = instance.get_physical_device_properties(physical_device);
    if properties.device_type != vk::PhysicalDeviceType::DISCRETE_GPU {
        return Err(anyhow!(SuitabilityError("Only discrete GPUs are supported.")));
    }

    let features = instance.get_physical_device_features(physical_device);
    if features.geometry_shader != vk::TRUE {
        return Err(anyhow!(SuitabilityError("Missing geometry shader support.")));
    }

    Ok(())
}
```

相比于直接选择第一个合适的设备，你也可以给每个设备评分，然后选择得分最高的那个。这样你就可以通过给独立显卡一个更高的分数来优先选择独立显卡，但是如果只有集成显卡可用，就回退到集成显卡。你也可以直接显示设备的名称，然后让用户自行选择。

接下来，我们会讨论我们第一个真正需要的功能。

## 队列族

之前已经介绍过，在 Vulkan 中进行任何操作（从绘制到纹理上传）基本都要将指令提交到队列。不同的队列族能够产生不同种类的队列，而每个队列族都只支持一部分指令。例如，可能有一个队列族只允许处理计算指令，或者只允许处理内存传输相关的指令。

我们需要查询设备支持的队列族，并且找到一个支持我们所需指令的队列族。为此，我们添加一个新的结构体 `QueueFamilyIndices` 来存储我们需要的队列族的索引。

<!-- 作者好菜 -->
现在，我们只要找到一个支持图形指令的队列族就好了，那么结构体和它的 `impl` 块看起来就像这样：

```rust,noplaypen
#[derive(Copy, Clone, Debug)]
struct QueueFamilyIndices {
    graphics: u32,
}

impl QueueFamilyIndices {
    unsafe fn get(
        instance: &Instance,
        data: &AppData,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Self> {
        let properties = instance
            .get_physical_device_queue_family_properties(physical_device);

        let graphics = properties
            .iter()
            .position(|p| p.queue_flags.contains(vk::QueueFlags::GRAPHICS))
            .map(|i| i as u32);

        if let Some(graphics) = graphics {
            Ok(Self { graphics })
        } else {
            Err(anyhow!(SuitabilityError("Missing required queue families.")))
        }
    }
}
```

`get_physical_device_queue_familiy_properties` 返回的队列属性包含了许多关于物理设备支持的队列族的细节，包括队列族支持的操作类型，以及基于这个队列族能创建多少队列。这里我们要找到第一个支持图形操作的队列族，这个队列族的标志是 `vk::QueueFlags::GRAPHICS`。

有了这个酷毙了的队列族查询方法，我们就可以在 `check_physical_device` 函数中使用它，来检查物理设备是否能够处理我们想要使用的指令：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    QueueFamilyIndices::get(instance, data, physical_device)?;
    Ok(())
}
```

<!-- 作者真的好菜 -->
最后，我们遍历所有物理设备，并选中第一个通过 `check_physical_device` 函数检测、符合我们要求的设备。我们更新 `pick_physical_device` 函数：

```rust,noplaypen
unsafe fn pick_physical_device(instance: &Instance, data: &mut AppData) -> Result<()> {
    for physical_device in instance.enumerate_physical_devices()? {
        let properties = instance.get_physical_device_properties(physical_device);

        if let Err(error) = check_physical_device(instance, data, physical_device) {
            warn!("Skipping physical device (`{}`): {}", properties.device_name, error);
        } else {
            info!("Selected physical device (`{}`).", properties.device_name);
            data.physical_device = physical_device;
            return Ok(());
        }
    }

    Err(anyhow!("Failed to find suitable physical device."))
}
```

好极了，这就是我们找到正确的物理设备所需要的一切！下一步是创建一个逻辑设备来与之交互。
