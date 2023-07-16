# Logical device and queues

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/logical_device_and_queues.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/04_logical_device.rs)

选择了要使用的物理设备之后，我们需要设置一个与之交互的逻辑设备。逻辑设备的创建过程与实例的创建过程相似，即描述我们希望使用的功能。既然我们已经查询了可用的队列族，我们还需要指定要创建哪些队列。如果你有不同的需求，甚至可以从同一个物理设备创建多个逻辑设备。

首先，在 `App` 中添加一个新的字段来存储逻辑设备：

```rust,noplaypen
struct App {
    // ...
    device: Device,
}
```

接下来，在 `App::create` 中调用 `create_logical_device` 函数，并将得到的逻辑设备添加到 `App` 的初始化器（initializer）中：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        let device = create_logical_device(&entry, &instance, &mut data)?;
        Ok(Self { entry, instance, data, device })
    }
}

unsafe fn create_logical_device(
    entry: &Entry,
    instance: &Instance,
    data: &mut AppData,
) -> Result<Device> {
}
```

## 指定要创建的队列

创建逻辑设备需要再用结构体指定一堆细节，其中第一个是 `vk::DeviceQueueCreateInfo`。这个结构体描述了我们为单个队列族需要的队列数量。现在，我们只对具备图形功能的队列感兴趣。

```rust,noplaypen
let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;

let queue_priorities = &[1.0];
let queue_info = vk::DeviceQueueCreateInfo::builder()
    .queue_family_index(indices.graphics)
    .queue_priorities(queue_priorities);
```

当前可用的驱动程序只允许你为每个队列族创建少量队列，实际上你也确实不需要多个队列。那是因为你可以在多个线程上创建命令缓冲区，然后在主线程上一次性提交它们，这样只需要一次低开销的调用。

Vulkan允许你为队列分配优先级，使用介于 `0.0` 和 `1.0` 之间的浮点数来影响命令缓冲区执行的调度。即使只创建一个队列，也需要指定优先级。

## 指定要启用的层

接下来要提供的信息与 `vk::InstanceCreateInfo` 结构体相似。同样的，我们需要指定要启用的任何层或扩展，但这次指定的扩展是设备特定的，而不是全局的。

一个设备特定的扩展的例子是 `VK_KHR_swapchain`，它允许您将从该设备渲染的图像呈现到窗口中。系统中可能存在一些不具备此功能的 Vulkan 设备，例如因为它们仅支持计算操作。我们将在交换链章节中再次提到这个扩展。

Vulkan以前的实现区分了实例和设备特定的验证层，但现在不再是这种情况。这意味着我们传递给 `enabled_layer_names` 的层名将被最新的实现忽略。不过，为了与旧版本兼容，还是应该设置这些名称。

我们还不会启用任何设备扩展。因此，如果启用了验证，我们将构建一个包含验证层的层名列表。

```rust,noplaypen
let layers = if VALIDATION_ENABLED {
    vec![VALIDATION_LAYER.as_ptr()]
} else {
    vec![]
};
```

## 指定要启用的扩展

正如 `实例` 一章中所讨论的，对于使用不完全符合 Vulkan 规范的 Vulkan 实现的应用程序，必须启用某些 Vulkan 扩展。在本章中，我们启用了与这些不符合规范的实现兼容所需的实例扩展。在这里，我们将启用出于同样目的所需的设备扩展。

```rust,noplaypen
let mut extensions = vec![];

// Required by Vulkan SDK on macOS since 1.3.216.
if cfg!(target_os = "macos") && entry.version()? >= PORTABILITY_MACOS_VERSION {
    extensions.push(vk::KHR_PORTABILITY_SUBSET_EXTENSION.name.as_ptr());
}
```

## 指定使用的设备功能

下一个需要指定的信息是我们将要使用的设备特性集。这些特性是我们在上一章中通过 `get_physical_device_features` 查询到的，比如几何着色器。现在我们不需要任何特殊的东西，所以我们可以简单地定义它，并将所有东西都保留为默认值（`false`）。一旦我们要开始使用Vulkan做更有趣的事情，我们会再回到这个结构。

```rust,noplaypen
let features = vk::PhysicalDeviceFeatures::builder();
```

## 创建逻辑设备

有了前面两个结构体、启用的验证层（如果启用）以及设备扩展，我们可以填充主要的 `vk::DeviceCreateInfo` 结构体。

```rust,noplaypen
let queue_infos = &[queue_info];
let info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(queue_infos)
    .enabled_layer_names(&layers)
    .enabled_extension_names(&extensions)
    .enabled_features(&features);
```

就是这样，我们现在可以调用名为 `create_device` 的方法来实例化逻辑设备了。

```rust,noplaypen
let device = instance.create_device(data.physical_device, &info, None)?;
```

参数是要与之交互的物理设备，我们刚刚指定的队列和使用信息，以及可选的分配回调。与实例创建函数类似，如果启用不存在的扩展或指定了不支持的功能，则此调用可能会返回错误。

设备应在 `App::destroy` 中被销毁：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_device(None);
    // ...
}
```

逻辑设备不直接与实例交互，因此不作为参数。

## 检索队列句柄

队列会随着逻辑设备的创建而自动创建，但我们还没有接口与它们交互的句柄。首先，在 `AppData` 中添加一个新的字段来存储图形队列的句柄：

```rust,noplaypen
struct AppData {
    // ...
    graphics_queue: vk::Queue,
}
```

设备队列会在设备销毁时自动清理，所以我们不需要在 `App::destroy` 中做任何处理。

我们可以使用 `get_device_queue` 函数来检索每个队列族的队列句柄。参数是逻辑设备、队列族和队列序号。因为我们只从该族中创建一个队列，所以我们只需使用序号 0。

```rust,noplaypen
data.graphics_queue = device.get_device_queue(indices.graphics, 0);
```

最后，在 `create_logical_device` 中返回创建的逻辑设备：

```rust,noplaypen
Ok(device)
```

有了逻辑设备和队列句柄，我们现在就可以真正开始使用显卡（graphics card）来执行任务了！在接下来的几章中，我们将设置资源以将结果呈现给窗口系统。
