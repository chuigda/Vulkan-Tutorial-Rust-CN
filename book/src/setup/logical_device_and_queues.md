# Logical device and queues

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/logical_device_and_queues.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/04_logical_device.rs)

选择了要使用的物理设备之后，我们需要设置一个与之交互的逻辑设备。逻辑设备的创建过程与实例的创建过程相似，即描述我们希望使用的功能。我们还需要指定要创建的队列，现在我们已经查询了可用的队列族。如果您有不同的需求，甚至可以从同一个物理设备创建多个逻辑设备。

首先，在 `App` 中添加一个新的字段来存储逻辑设备：

```rust,noplaypen
struct App {
    // ...
    device: Device,
}
```

接下来，在 `App::create` 中调用 `create_logical_device` 函数，并将得到的逻辑设备添加到 `App` 的初始化器中：

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

创建逻辑设备涉及再次在结构体中指定一堆细节，其中第一个是 `vk::DeviceQueueCreateInfo`。这个结构体描述了我们为单个队列族需要的队列数量。现在，我们只对具备图形功能的队列感兴趣。

```rust,noplaypen
let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;

let queue_priorities = &[1.0];
let queue_info = vk::DeviceQueueCreateInfo::builder()
    .queue_family_index(indices.graphics)
    .queue_priorities(queue_priorities);
```

当前可用的驱动程序只允许您为每个队列族创建少量队列，而实际上您并不需要多个队列。这是因为您可以在多个线程上创建所有命令缓冲区，然后在主线程上一次性提交它们，只需要一次低开销的调用。

Vulkan允许您为队列分配优先级，使用介于 `0.0` 和 `1.0` 之间的浮点数来影响命令缓冲区执行的调度。即使只创建一个队列，也需要指定优先级。

## 指定要启用的层

我们需要提供的下一个信息与 `vk::InstanceCreateInfo` 结构体相似。再次，我们需要指定要启用的任何层或扩展，但这次指定的扩展是设备特定的，而不是全局的。

一个设备特定扩展的例子是 `VK_KHR_swapchain`，它允许您将从该设备渲染的图像呈现到窗口。可能存在系统中缺少此功能的Vulkan设备，例如因为它们只支持计算操作。我们将在交换链一章中再次提到这个扩展。

先前的Vulkan实现区分了实例和设备特定的验证层，但这种情况不再存在。这意味着我们稍后传递给 `enabled_layer_names` 的层名称在最新的实现中将被忽略。然而，为了与旧的实现兼容，仍然将它们设置为一个好主意。

我们暂时还不会启用任何设备扩展，所以我们只需构造一个包含验证层的层名称列表，如果启用了验证。

```rust,noplaypen
let layers = if VALIDATION_ENABLED {
    vec![VALIDATION_LAYER.as_ptr()]
} else {
    vec![]
};
```

## 指定要启用的扩展

如在 `实例` 章节所讨论的，对于使用与 Vulkan 规范不完全符合的 Vulkan 实现的应用程序，必须启用某些 Vulkan 扩展。在那一部分中，我们启用了与这些不符合规范的实现兼容所需的实例扩展。在这里，我们将启用同样目的的设备扩展。

```rust,noplaypen
let mut extensions = vec![];

// Required by Vulkan SDK on macOS since 1.3.216.
if cfg!(target_os = "macos") && entry.version()? >= PORTABILITY_MACOS_VERSION {
    extensions.push(vk::KHR_PORTABILITY_SUBSET_EXTENSION.name.as_ptr());
}
```

## 指定使用的设备功能

接下来要指定的是我们将使用的设备功能集。这些是我们在上一章中使用 `get_physical_device_features` 查询支持的功能，如几何着色器。现在我们不需要任何特殊功能，所以我们可以简单地定义它并将所有值保留为默认值（`false`）。在我们开始使用 Vulkan 进行更有趣的事情之前，我们将回到这个结构体。

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

现在，我们准备好通过调用适当命名的 `create_device` 方法来实例化逻辑设备了。

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

逻辑设备不直接与实例交互，这就是为什么它不作为参数传递的原因。

## 检索队列句柄

队列会随着逻辑设备的创建而自动创建，但我们还没有接口与它们交互的句柄。首先，在 `AppData` 中添加一个新的字段来存储图形队列的句柄：

```rust,noplaypen
struct AppData {
    // ...
    graphics_queue: vk::Queue,
}
```

设备队列会在设备销毁时自动清理，所以我们不需要在 `App::destroy` 中做任何处理。

我们可以使用 `get_device_queue` 函数来检索每个队列族的队列句柄。参数是逻辑设备、队列族和队列索引。因为我们只从该族中创建一个队列，所以我们将简单地使用索引 0。

```rust,noplaypen
data.graphics_queue = device.get_device_queue(indices.graphics, 0);
```

最后，在 `create_logical_device` 中返回创建的逻辑设备：

```rust,noplaypen
Ok(device)
```

有了逻辑设备和队列句柄，我们现在可以真正开始使用显卡来执行任务了！在接下来的几章中，我们将设置资源以将结果呈现给窗口系统。
