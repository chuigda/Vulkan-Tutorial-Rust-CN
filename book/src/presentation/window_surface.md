# 窗口表面

**代码：**[main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/05_window_surface.rs)

由于Vulkan是一个与平台无关的API，它不能直接与窗口系统进行交互。为了在屏幕上呈现结果，我们需要使用扩展`WSI（Window System Integration）`来建立Vulkan与窗口系统之间的连接。在本章中，我们将讨论第一个扩展，即`VK_KHR_surface`。它公开了一个`vk::SurfaceKHR`对象，表示一种用于呈现渲染图像的抽象表面。我们程序中的表面将由我们使用`winit`打开的窗口来支持。
`VK_KHR_surface`扩展是一个实例级扩展，我们实际上已经启用了它，因为它被包含在`vk_window::get_required_instance_extensions`返回的列表中。该列表还包括我们将在接下来的几章中使用的其他WSI扩展。

窗口表面需要在实例创建之后立即创建，因为它实际上可以影响物理设备的选择。我们之所以推迟创建窗口表面，是因为窗口表面是渲染目标和呈现的更大主题的一部分，如果在基本设置中解释这一点，会使得解释变得混乱。还应该注意的是，窗口表面在Vulkan中是一个完全可选的组件，如果你只需要离屏渲染，Vulkan允许你在不像OpenGL那样创建一个不可见窗口的情况下进行渲染。

虽然我们可以自由导入扩展类型，如结构体`vk::SurfaceKHR`，但在调用扩展添加的任何Vulkan命令之前，我们需要导入`vulkanalia`扩展特性`vk::KhrSurfaceExtension`。请添加以下导入语句以导入`vk::KhrSurfaceExtension`：

```rust,noplaypen
use vulkanalia::vk::KhrSurfaceExtension;
```

## 创建窗口表面

首先，在其他字段之前，向`AppData`中添加一个`surface`字段：

```rust,noplaypen
struct AppData {
    surface: vk::SurfaceKHR,
    // ...
}
```

虽然`vk::SurfaceKHR`对象及其使用是与平台无关的，但它的创建不是，它依赖于窗口系统的细节。例如，在Windows上，它需要`HWND`和`HMODULE`句柄。因此，扩展中有一个特定于平台的附加部分，在Windows上称为`VK_KHR_win32_surface`，它也会自动包含在`vk_window::get_required_instance_extensions`的列表中。

我将演示如何在Windows上使用这个特定于平台的扩展来创建表面，但实际上在本教程中我们不会使用它。`vulkanalia`提供了`vk_window::create_surface`，它可以处理平台差异。然而，在我们开始依赖它之前，了解它在幕后的工作原理是很有好处的。

因为窗口表面是一个Vulkan对象，所以它带有一个需要填充的`vk::Win32SurfaceCreateInfoKHR`结构体。它有两个重要的参数：`hinstance`和`hwnd`,分别是进程和窗口的句柄。

```rust,noplaypen
use winit::platform::windows::WindowExtWindows;

let info = vk::Win32SurfaceCreateInfoKHR::builder()
    .hinstance(window.hinstance())
    .hwnd(window.hwnd());
```

`WindowExtWindows`特性是从`winit`中导入的，因为它允许我们在`winit`的`Window`结构体上访问特定于平台的方法。在这种情况下，它允许我们获取由`winit`创建的窗口的进程和窗口的句柄。


之后可以使用`create_win32_surface_khr`创建表面，该函数包括用于表面创建的详细信息和自定义分配器的参数。从技术上讲，这是一个WSI扩展函数，但它如此常用，以至于标准的Vulkan加载器包含了它，因此不像其他扩展一样需要显式加载它。但是，我们确实需要导入`vulkanalia`扩展特性`VK_KHR_win32_surface`（`vk::KhrWin32SurfaceExtension`）。


```rust,noplaypen
use vk::KhrWin32SurfaceExtension;

let surface = instance.create_win32_surface_khr(&info, None).unwrap();
```

对于其他平台（如Linux），过程类似，在Linux上使用`create_xcb_surface_khr`函数，该函数接受XCB连接和窗口作为创建细节，并使用X11进行操作

`vk_window::create_surface`函数使用不同的实现执行完全相同的操作，具体取决于平台。现在，我们将其集成到我们的程序中。在`App::create`中，在选择物理设备之前，调用该函数：

```rust,noplaypen
unsafe fn create(window: &Window) -> Result<Self> {
    // ...
    let instance = create_instance(window, &entry, &mut data)?;
    data.surface = vk_window::create_surface(&instance, &window, &window)?;
    pick_physical_device(&instance, &mut data)?;
    // ...
}
```

参数是Vulkan实例和`winit`窗口。一旦我们有了表面，就可以在`App::destroy`中使用Vulkan API销毁它

```rust,noplaypen
unsafe fn destroy(&mut self) {
    // ...
    self.instance.destroy_surface_khr(self.data.surface, None);
    self.instance.destroy_instance(None);
}
```

确保在销毁实例之前销毁表面。

## 查询呈现支持

尽管Vulkan的实现可能支持窗口系统集成，但这并不意味着系统中的每个设备都支持。因此，我们需要扩展`QueueFamilyIndices`结构体，以确保设备能够向我们创建的表面呈现图像。由于呈现是与队列相关的功能，实际上问题是找到一个支持向我们创建的表面进行呈现的队列族。

实际上，支持绘制命令的队列族和支持呈现的队列族可能并不重叠。因此，我们必须考虑到可能存在不同的呈现队列，通过修改`QueueFamilyIndices`结构体来解决此问题：

```rust,noplaypen
struct QueueFamilyIndices {
    graphics: u32,
    present: u32,
}
```

接下来，我们将修改`QueueFamilyIndices::get`方法，以查找具有向我们的窗口表面进行呈现的能力的队列族，该方法使用`get_physical_device_surface_support_khr`函数，它以物理设备、队列族索引和表面为参数，并返回是否支持该组合的物理设备、队列族和表面的呈现。

修改`QueueFamilyIndices::get`，以查找绘制队列族索引下方的呈现队列族索引。

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

我们还需要将`present`添加到最终的表达式中：

```rust,noplaypen
if let (Some(graphics), Some(present)) = (graphics, present) {
    Ok(Self { graphics, present })
} else {
    Err(anyhow!(SuitabilityError("Missing required queue families.")))
}
```

请注意，这两个索引最终很可能是相同的队列族，但在整个程序中，我们将把它们视为独立的队列，以实现统一的方法。然而，你可以添加逻辑来明确优先选择支持绘制和呈现的相同队列的物理设备，以提高性能。

## 创建呈现队列

剩下的一件事是修改逻辑设备的创建过程，以创建呈现队列并检索`vk::Queue`句柄。在`AppData`中添加一个字段来保存句柄：

```rust,noplaypen
struct AppData {
    // ...
    present_queue: vk::Queue,
}
```

接下来，我们需要创建多个`vk::DeviceQueueCreateInfo`结构体来从两个队列族中创建队列。一种简单的方法是创建一个包含所有所需队列的唯一队列族集合。我们将在`create_logical_device`函数中完成这个操作：

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

然后删除之前的`queue_infos`切片，并为`vk::DeviceCreateInfo`引用`queue_infos`列表：

```rust,noplaypen
let info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(&queue_infos)
    .enabled_layer_names(&layers)
    .enabled_features(&features);
```

如果队列族相同，则只需传递一次索引。最后，添加一个调用来获取队列句柄：

```rust,noplaypen
data.present_queue = device.get_device_queue(indices.present, 0);
```

如果队列族相同，那么现在这两个句柄很可能具有相同的值。在下一章中，我们将讨论交换链以及它们如何使我们能够向表面呈现图像。