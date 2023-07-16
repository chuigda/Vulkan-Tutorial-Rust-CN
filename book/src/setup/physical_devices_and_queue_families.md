# 物理设备与队列族

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/physical_devices_and_queue_families.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/03_physical_device_selection.rs)

在通过 `Instance` 初始化 Vulkan 库之后，我们需要在系统中选择一个支持我们所需功能的图形处理器。事实上，我们可以选择任意多个图形处理器，并同时使用它们，不过在本教程中我们只会选择第一个满足我们需求的图形处理器。

我们将添加一个名为 `pick_physical_device` 的函数来完成这个任务，并将物理设备及其相关信息写入 `AppData` 实例。这个函数及其调用的函数将使用自定义错误类型（`SuitabilityError`）来表示物理设备不满足应用程序的要求。这个错误类型将使用 `thiserror` crate 来自动实现所有必要的错误类型的样板代码。

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

我们将选择的显卡将存储在 `vk::PhysicalDevice` 句柄中，并作为 `AppData` 结构体的一个新字段添加。这个对象将在 `Instance` 被销毁时被隐式销毁，因此我们不需要在 `App::destroy` 方法中做任何新的操作。

```rust,noplaypen
struct AppData {
    // ...
    physical_device: vk::PhysicalDevice,
}
```

## 设备适用性

我们需要一种方法来确定一个物理设备是否符合我们的需求。我们将首先创建一个函数，如果提供的物理设备不支持我们需要的所有功能，则返回一个 `SuitabilityError`：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    Ok(())
}
```

为了评估一个物理设备是否符合我们的需求，我们可以从设备中查询一些详细信息。可以使用 `get_physical_device_properties` 查询设备的基本属性，比如名称、类型和支持的Vulkan版本：

```rust,noplaypen
let properties = instance
    .get_physical_device_properties(physical_device);
```

可以使用 `get_physical_device_features` 查询对可选功能的支持，比如纹理压缩、64 位浮点数和多视口渲染（在VR中很有用）：

```rust,noplaypen
let features = instance
    .get_physical_device_features(physical_device);
```

还有更多关于设备的细节可以查询，我们将在后面讨论与设备内存和队列族相关的内容（请参阅下一节）。

举个例子，假设我们认为我们的应用程序只适用于支持几何着色器的独立显卡，那么 `check_physical_device` 函数可能如下所示：

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

与其仅仅检查设备是否适合或不适合，并选择第一个设备，您还可以为每个设备评分并选择得分最高的设备。这样，您可以优先选择独立显卡，并给它一个较高的分数，但如果只有集成显卡可用，那么可以回退到集成显卡。您还可以只显示可用设备的名称，并允许用户进行选择。

接下来，我们将讨论第一个真正需要的功能。

## Queue families

It has been briefly touched upon before that almost every operation in Vulkan, anything from drawing to uploading textures, requires commands to be submitted to a queue. There are different types of queues that originate from different queue families and each family of queues allows only a subset of commands. For example, there could be a queue family that only allows processing of compute commands or one that only allows memory transfer related commands.

We need to check which queue families are supported by the device and which one of these supports the commands that we want to use. For that purpose we'll add a new struct `QueueFamilyIndices` that stores the indices of the queue families we need.

Right now we are only going to look for a queue that supports graphics commands, so the struct and its implementation will look like this:

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

The queue properties returned by `get_physical_device_queue_family_properties` contains various details about the queue families supported by the physical device, including the type of operations supported and the number of queues that can be created based on that family. Here we are looking for the first queue family that supports graphics operations as indicated by `vk::QueueFlags::GRAPHICS`.

Now that we have this fancy queue family lookup method, we can use it as a check in the `check_physical_device` function to ensure the device can process the commands we want to use:

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

Lastly we can iterate over the physical devices and pick the first that satisfies our requirements as indicated by `check_physical_device`. To do this, update `pick_physical_device` to look like the following:

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

Great, that's all we need for now to find the right physical device! The next step is to create a logical device to interface with it.
