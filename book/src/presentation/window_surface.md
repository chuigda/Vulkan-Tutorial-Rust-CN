# 窗口表面

> 原文链接：<https://kylemayes.github.io/vulkanalia/presentation/window_surface.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码：**[main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/05_window_surface.rs)

Vulkan 是一个平台无关的 API，因此它不能直接与窗口系统进行交互。要在屏幕上呈现结果，我们需要使用一系列 WSI（Window System Interface，窗口系统接口）扩展来建立 Vulkan 与窗口系统之间的连接。在本章中，我们将讨论第一个扩展，即 `VK_KHR_surface`。它暴露了一个 `vk::SurfaceKHR` 类型，表示一种用于呈现图像的抽象表面。我们程序中的窗口表面将由我们用 `winit` 打开的窗口来支持。

`VK_KHR_surface` 扩展是一个实例级扩展，我们实际上已经启用了它，因为它被包含在 `vk_window::get_required_instance_extensions` 返回的列表中。该列表还包含了我们将在接下来的几章中使用的其他 WSI 扩展。

窗口表面需要在创建实例之后立即创建，因为窗口表面实际上会影响物理设备的选择。我们之所以现在才讲窗口表面的创建，是因为窗口表面是“渲染目标与呈现”这一更大主题的一部分，在“基本设置”那部分里解释窗口表面会引起混乱。此外还要注意，窗口表面是 Vulkan 里一个完全可选的组件，如果你只需要离屏渲染，Vulkan 也可以在不创建窗口的情况下进行渲染（而 OpenGL 就必须用创建一个不可见窗口这种投机取巧的方式）。

<!-- 作者是真的逆天 -->
要使用 `VK_KHR_surface` 扩展，我们除了要导入 `vk::SurfaceKHR` 类型，还需要导入 `vulkanalia` 的扩展 trait `vk::KhrSurfaceExtension`：

```rust,noplaypen
use vulkanalia::vk::SurfaceKHR
use vulkanalia::vk::KhrSurfaceExtension;
```

## 创建窗口表面

首先，在 `AppData` 中，在**其他字段的上面**添加一个 `surface` 字段：

```rust,noplaypen
struct AppData {
    surface: vk::SurfaceKHR,
    // ...
}
```

虽然 `vk::SurfaceKHR` 对象及其用法是与平台无关的，但创建它的过程不是，创建 `vk::SurfaceKHR` 的具体过程依赖于窗口系统的细节。例如在 Windows 上，创建 `vk::SurfaceKHR` 需要 `HWND` 和 `HMODULE` 句柄。因此，扩展中有一个特定于平台的附加部分，例如在 Windows 上，它是 `VK_KHR_win32_surface`。平台特定的附加部分也会被自动包含在 `vk_window::get_required_instance_extensions` 的列表中。

我将演示如何在 Windows 上使用这个特定于平台的扩展来创建表面，但实际上在本教程中我们不会使用它。`vulkanalia` 已经提供了 `vk_window::create_surface`，它可以处理平台之间的差异。不过在我们开始使用它之前，了解幕后的工作原理是很有好处的。

因为窗口表面是一个 Vulkan 对象，所以和其他 Vulkan 对象一样，创建它需要填充一个的 `vk::Win32SurfaceCreateInfoKHR` 结构体。它有两个重要的参数：`hinstance` 和 `hwnd`，分别是进程和窗口的句柄。

```rust,noplaypen
use winit::platform::windows::WindowExtWindows;

let info = vk::Win32SurfaceCreateInfoKHR::builder()
    .hinstance(window.hinstance())
    .hwnd(window.hwnd());
```

`WindowExtWindows` 特性是从 `winit` 中导入的，它允许我们在 `winit` 的 `Window` 结构体上访问平台特定的方法。在这种情况下，它允许我们获取由 `winit` 创建的窗口所在进程的句柄（`hinstance`）和窗口的句柄（`hwnd`）。

之后使用 `create_win32_surface_khr` 创建表面，该函数包括用于表面创建的详细信息和自定义分配器的参数。从技术上讲，这是一个 WSI 扩展函数，但它的使用频率很高，所以标准的 Vulkan 加载器也会加载它，因而它不需要像其他扩展一样显式加载。不过我们还是需要为扩展 `VK_KHR_win32_surface` 导入 `vulkanalia` 的扩展 trait `vk::KhrWin32SurfaceExtension`。

```rust,noplaypen
use vk::KhrWin32SurfaceExtension;

let surface = instance.create_win32_surface_khr(&info, None).unwrap();
```

在其他平台（如 Linux）上创建表面的过程也和上面类似。例如在 Linux 上需要使用 `create_xcb_surface_khr` 函数，该函数接受 XCB 连接和窗口，并在背后调用 X11 的 API。

`vk_window::create_surface` 函数在不同的平台上使用不同的实现执行完全相同的操作。现在，我们将其集成到程序中。在 `App::create` 中，在选择物理设备之前，调用该函数：

```rust,noplaypen
unsafe fn create(window: &Window) -> Result<Self> {
    // ...
    let instance = create_instance(window, &entry, &mut data)?;
    data.surface = vk_window::create_surface(&instance, &window, &window)?;
    pick_physical_device(&instance, &mut data)?;
    // ...
}
```

参数是 Vulkan 实例和 `winit` 窗口。一旦我们创建了表面，我们就需要在 `App::destroy` 中使用 Vulkan API 销毁它：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    // ...
    self.instance.destroy_surface_khr(self.data.surface, None);
    self.instance.destroy_instance(None);
}
```

确保在销毁实例之前销毁表面。

## 查询呈现（presentation）支持

<!-- 这里是故意没按照原文翻译的，因为原文真把我气笑了。之后可能会给原文提 PR 以修正这个问题。 -->
尽管 Vulkan 的实现可能支持窗口系统集成，但这并不意味着系统中的每个设备都支持。因此，我们需要扩展 `pick_physical_device` 函数的功能，以确保我们选择的设备能够向我们创建的表面呈现图像。因为呈现是与队列相关的功能，所以我们实际上是要找到一个支持向我们创建的表面进行呈现的队列族。

事实上，支持绘制指令的队列族和支持呈现的队列族可能并不重叠。因此，我们必须考虑呈现队列不同于图形队列的可能性，并修改 `QueueFamilyIndices` 结构体来解决此问题：

```rust,noplaypen
struct QueueFamilyIndices {
    graphics: u32,
    present: u32,
}
```

接下来，我们将修改 `QueueFamilyIndices::get` 方法，以查找能向我们的窗口表面进行呈现的队列族。该方法使用 `get_physical_device_surface_support_khr` 函数，它以物理设备、队列族索引和表面为参数，并返回这个物理设备、队列族和表面的组合是否支持呈现：

```rust,noplaypen
let mut present = None;
for (index, properties) in properties.iter().enumerate() {
    if instance.get_physical_device_surface_support_khr(
        physical_device,
        index as u32,
        data.surface,
    )? {
        present = Some(index as u32);
        break;
    }
}
```

我们还需要将 `present` 添加到最终的表达式中：

```rust,noplaypen
if let (Some(graphics), Some(present)) = (graphics, present) {
    Ok(Self { graphics, present })
} else {
    Err(anyhow!(SuitabilityError("Missing required queue families.")))
}
```

请注意，这两个索引最终很可能指涉到相同的队列族，但在整个程序中，我们将把它们视为独立的队列，这样我们就可以用统一的方式来处理它们。你也可以添加逻辑来优先选择能在同一个队列中进行绘制和呈现的物理设备，以提高性能。

## 创建呈现队列

最后一件事是修改逻辑设备的创建过程，以创建呈现队列并取得其 `vk::Queue` 句柄。在 `AppData` 中添加一个字段来保存呈现队列的句柄：

```rust,noplaypen
struct AppData {
    // ...
    present_queue: vk::Queue,
}
```

接下来，我们需要创建多个 `vk::DeviceQueueCreateInfo` 结构体来从两个队列族中创建队列。一种简单的方法是创建一个集合，用来去重并保存所有需要的队列族。我们将在 `create_logical_device` 函数中完成这个操作：

```rust,noplaypen
let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;

let mut unique_indices = HashSet::new();
unique_indices.insert(indices.graphics);
unique_indices.insert(indices.present);

let queue_priorities = &[1.0];
let queue_infos = unique_indices
    .iter()
    .map(|i| {
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(*i)
            .queue_priorities(queue_priorities)
    })
    .collect::<Vec<_>>();
```

然后删除之前的 `queue_infos` 切片，并为 `vk::DeviceCreateInfo` 提供一个 `queue_infos` 列表的引用：

```rust,noplaypen
let info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(&queue_infos)
    .enabled_layer_names(&layers)
    .enabled_features(&features);
```

最后，添加一个调用来获取队列句柄：

```rust,noplaypen
data.present_queue = device.get_device_queue(indices.present, 0);
```

如果队列族相同，那么现在这两个句柄很可能具有相同的值。在下一章中，我们将讨论交换链以及它们如何使我们能够向表面呈现图像。
