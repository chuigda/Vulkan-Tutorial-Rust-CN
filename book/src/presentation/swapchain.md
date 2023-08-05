# 交换链

> 原文链接：<https://kylemayes.github.io/vulkanalia/presentation/swapchain.html>
>
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/06_swapchain_creation.rs)

Vulkan 没有“默认帧缓冲”（default framebuffer）的概念，因此，Vulkan 需要一个结构来持有我们将要绘制的帧缓冲，这个架构就是*交换链*。在 Vulkan 中，交换链必须被显式创建。交换链本质上就是一个队列，其中充满了等待呈现到屏幕上的图像。我们的应用程序每次会从这个队列中获取一张图像，在上面绘制，然后将它返还到队列中。交换链的设置决定了这个队列如何工作，以及何时呈现队列中的图像，但通常来说，交换链的目的是使图像的呈现与屏幕刷新率同步。

## 检测交换链支持

出于某些原因，不是所有的显卡都能直接向屏幕呈现图像，例如有些显卡是为服务器设计的，没有图像输出接口。其次，呈现图像和窗口系统以及与窗口系统关联的表面密切相关。因此，交换链不是 Vulkan 核心的一部分。你必须先查询设备对交换链扩展 `VK_KHR_swapchain` 的支持，然后启用它。

像之前一样，我们首先导入 `vulkanalia` 的扩展 trait `vk::KhrSwapchainExtension`：

```rust,noplaypen
use vulkanalia::vk::KhrSwapchainExtension;
```

接着，我们扩展 `check_physical_device` 函数，增加对 `VK_KHR_swapchain` 扩展支持的检查。我们之前已经看过如何列出一个物理设备支持的扩展，所以这一步应该非常直观。

首先声明一个所需设备扩展的列表，这一步和启用校验层的列表类似：

```rust,noplaypen
const DEVICE_EXTENSIONS: &[vk::ExtensionName] = &[vk::KHR_SWAPCHAIN_EXTENSION.name];
```

然后创建一个新函数 `check_physical_device_extensions` 作为 `check_physical_device` 的附加检查：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    QueueFamilyIndices::get(instance, data, physical_device)?;
    check_physical_device_extensions(instance, physical_device)?;
    Ok(())
}

unsafe fn check_physical_device_extensions(
    instance: &Instance,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    Ok(())
}
```

修改 `check_physical_device_extensions` 的函数体，枚举设备支持的所有扩展，并检查其中是否包含所有所需的扩展：

```rust,noplaypen
unsafe fn check_physical_device_extensions(
    instance: &Instance,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    let extensions = instance
        .enumerate_device_extension_properties(physical_device, None)?
        .iter()
        .map(|e| e.extension_name)
        .collect::<HashSet<_>>();
    if DEVICE_EXTENSIONS.iter().all(|e| extensions.contains(e)) {
        Ok(())
    } else {
        Err(anyhow!(SuitabilityError("Missing required device extensions.")))
    }
}
```

现在运行代码，确保你的显卡支持交换链的创建。值得注意的是，我们在前一章检查呈现队列的可用性时，已经隐式地检查了交换链扩展的支持。不过显式地检查一下也好，而且交换链扩展必须显式地启用。

## 启用设备扩展

使用交换链需要先启用 `VK_KHR_swapchain` 扩展。启用扩展只需要在 `create_logical_device` 函数中对设备扩展列表进行一点小小的修改。使用 `DEVICE_EXTENSIONS` 构造一个由空结尾的字符串组成的列表，来初始化我们的设备扩展列表：

```rust,noplaypen
let mut extensions = DEVICE_EXTENSIONS
    .iter()
    .map(|n| n.as_ptr())
    .collect::<Vec<_>>();
```

## 查询交换链支持的细节

只检查交换链是否可用还不够，因为它不一定和我们的窗口表面兼容。创建交换链还需要更多的设置，因此在继续推进之前，我们需要查询更多的细节。

总的来说，我们需要检查三种基本属性：

* 基本的表面能力（交换链中图像的最小/最大数量，图像的最小/最大宽度和高度）
* 表面格式（像素格式，颜色空间）
* 可用的呈现模式

与 `QueueFamilyIndices` 类似，我们会使用一个结构体来存储这些细节：

```rust,noplaypen
#[derive(Clone, Debug)]
struct SwapchainSupport {
    capabilities: vk::SurfaceCapabilitiesKHR,
    formats: Vec<vk::SurfaceFormatKHR>,
    present_modes: Vec<vk::PresentModeKHR>,
}
```

现在我们创建一个新的方法 `SwapchainSupport::get`，用来初始化这个结构体，填充我们所需的所有字段：

```rust,noplaypen
impl SwapchainSupport {
    unsafe fn get(
        instance: &Instance,
        data: &AppData,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Self> {
        Ok(Self {
            capabilities: instance
                .get_physical_device_surface_capabilities_khr(
                    physical_device, data.surface)?,
            formats: instance
                .get_physical_device_surface_formats_khr(
                    physical_device, data.surface)?,
            present_modes: instance
                .get_physical_device_surface_present_modes_khr(
                    physical_device, data.surface)?,
        })
    }
}
```

这些字段的含义以及它们包含的数据的确切含义将在下一节中讨论。

现在，所有细节都在这个结构体里了，让我们再扩展一次 `check_physical_device` 函数，用这个方法来验证交换链的支持是否足够。只要交换链支持至少一种图像格式，以及至少一种给定窗口表面的呈现模式，那么这个交换链就可以满足本教程的需求。

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    // ...

    let support = SwapchainSupport::get(instance, data, physical_device)?;
    if support.formats.is_empty() || support.present_modes.is_empty() {
        return Err(anyhow!(SuitabilityError("Insufficient swapchain support.")));
    }

    Ok(())
}
```

注意我们必须在确认交换链扩展可用之后，再检查交换链支持。

## 为交换链选择正确的设置

如果交换链满足我们刚刚所说的那些条件，那么这个交换链肯定是够用了。但交换链支持的设置很多，我们还需要做到最好。我们现在要写一些函数来找到最佳的交换链设置。有三种类型的设置需要确定：

* 表面格式（颜色深度）
* 呈现模式（将图像“交换”到屏幕的条件）
* 交换范围（swap extent）（交换链中图像的分辨率）

每一种设置都有一个理想值，如果这个理想值可用，我们就使用它，否则我们就创建一些逻辑来找到次佳的值。

### 表面格式

我们从一个下面这样的函数开始，稍后我们会把 `SwapchainSupport` 结构体的 `formats` 字段传给它作参数：

```rust,noplaypen
fn get_swapchain_surface_format(
    formats: &[vk::SurfaceFormatKHR],
) -> vk::SurfaceFormatKHR {
}
```

`vk::SurfaceFormatKHR` 有 `format` 和 `color_space` 两个成员。`format` 指定颜色的通道数和类型。例如，`vk::Format::B8G8R8A8_SRGB` 表示我们按照 B、G、R 和 alpha 通道的顺序存储颜色，每个通道使用 8 位无符号整数，每像素总共 32 位。`color_space` 成员使用 `vk::ColorSpaceKHR::SRGB_NONLINEAR` 标志表示是否支持 sRGB 颜色空间。

因为 sRGB 颜色空间可以[更准确地表示颜色](http://stackoverflow.com/questions/12524623/)，所以我们会优先使用它。它也是纹理等图像的标准颜色空间。因此，我们也应该优先使用 sRGB 颜色格式，其中最常见的一个就是 `vk::Format::B8G8R8A8_SRGB`。

让我们遍历 `formats` 列表，看看是否有我们想要的组合：

```rust,noplaypen
fn get_swapchain_surface_format(
    formats: &[vk::SurfaceFormatKHR],
) -> vk::SurfaceFormatKHR {
    formats
        .iter()
        .cloned()
        .find(|f| {
            f.format == vk::Format::B8G8R8A8_SRGB
                && f.color_space == vk::ColorSpaceKHR::SRGB_NONLINEAR
        })
        .unwrap_or_else(|| formats[0])
}
```

如果没有，那么我们可以评估可用格式的优劣，然后选择一个最好的。但在大多数情况下，随遇而安地使用第一个格式也行，所以我们用 `unwrap_or_else` 方法来简化代码。

### 呈现模式

呈现模式可以说是交换链中最重要的设置，因为它决定了图像什么时候被交换到屏幕上。Vulkan 中有四种可能的呈现模式：

* `vk::PresentModeKHR::IMMEDIATE` &ndash; 应用程序提交的图像会立即传输到屏幕上，这可能会导致撕裂。
* `vk::PresentModeKHR::FIFO` &ndash; 交换链是一个队列，当显示器刷新时，显示器会从队列的前端取出一张图像，应用程序会在队列的后端插入渲染好的图像。如果队列已满，应用程序就必须等待。这种模式最类似于现代游戏中的垂直同步（vertical sync）。显示器刷新的时刻被称为“垂直空白”（vertical blank）。
* `vk::PresentModeKHR::FIFO_RELAXED` &ndash; 这种模式与 `FIFO` 的区别在于，如果程序提交图像的速度比显示器刷新的速度慢，那么图像就会立即传输，而不是等待下一个垂直空白。这可能会导致撕裂。
* `vk::PresentModeKHR::MAILBOX` &ndash; 这是 `FIFO` 模式的另一种变体。如果程序提交图像的速度比显示器刷新的速度快，当队列已满时，队列中的图像会直接被新的图像替换，而不会阻塞应用程序。这种模式可以用来尽可能快地渲染帧，同时避免撕裂，因而比标准的垂直同步有更少的延迟。这通常被称为“三重缓冲”，尽管仅靠三个缓冲本身并不能让帧率不受限制。

只有 `vk::PresentModeKHR::FIFO` 模式是保证可用的，因此我们需要写一个函数来查找可用的最佳模式：

```rust,noplaypen
fn get_swapchain_present_mode(
    present_modes: &[vk::PresentModeKHR],
) -> vk::PresentModeKHR {
}
```

我个人认为，如果能耗不是问题的话，`vk::PresentModeKHR::MAILBOX` 是一个非常好的折中方案。它既能避免撕裂，同时又能保持尽可能低的延迟，因为它会在垂直空白之前渲染尽可能新的图像。在移动设备上，能耗更重要，那时候你可能会想使用 `vk::PresentModeKHR::FIFO`。现在，让我们遍历 `present_modes` 列表，看看 `vk::PresentModeKHR::MAILBOX` 是否可用：

```rust,noplaypen
fn get_swapchain_present_mode(
    present_modes: &[vk::PresentModeKHR],
) -> vk::PresentModeKHR {
    present_modes
        .iter()
        .cloned()
        .find(|m| *m == vk::PresentModeKHR::MAILBOX)
        .unwrap_or(vk::PresentModeKHR::FIFO)
}
```

### 交换范围

现在就剩交换范围一个属性了，我们再为它写一个函数：

```rust,noplaypen
fn get_swapchain_extent(
    window: &Window,
    capabilities: vk::SurfaceCapabilitiesKHR.
) -> vk::Extent2D {
}
```

交换范围就是交换链中图像的分辨率，它几乎总是等于我们正在绘制的窗口的分辨率。可用的分辨率范围在 `vk::SurfaceCapabilitiesKHR` 结构体中定义。Vulkan 通过 `current_extent` 成员来告知适合我们窗口的交换范围。一些窗口系统会将 `current_extent` 的宽和高设置为一个特殊值 —— `u32` 类型的最大值 —— 来表示允许我们自己选择对于窗口最合适的交换范围，在这种情况下我们需要在 `min_image_extent` 和 `max_image_extent` 的范围内选择一个最合适的分辨率。

```rust,noplaypen
fn get_swapchain_extent(
    window: &Window,
    capabilities: vk::SurfaceCapabilitiesKHR,
) -> vk::Extent2D {
    if capabilities.current_extent.width != u32::max_value() {
        capabilities.current_extent
    } else {
        let size = window.inner_size();
        let clamp = |min: u32, max: u32, v: u32| min.max(max.min(v));
        vk::Extent2D::builder()
            .width(clamp(
                capabilities.min_image_extent.width,
                capabilities.max_image_extent.width,
                size.width,
            ))
            .height(clamp(
                capabilities.min_image_extent.height,
                capabilities.max_image_extent.height,
                size.height,
            ))
            .build()
    }
}
```

我们使用 `clamp` 函数来限制窗口的实际大小在 Vulkan 设备支持的范围内。

## 创建交换链

现在我们有了所有用来帮助我们在运行时做出决策的辅助函数，我们终于有了创建工作交换链所需的所有信息。

创建一个 `create_swapchain` 函数，它首先调用这些辅助函数并取得其结果。然后，在 `App::create` 中创建逻辑设备之后调用这个函数：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        let device = create_logical_device(&instance, &mut data)?;
        create_swapchain(window, &instance, &device, &mut data)?;
        // ...
    }
}

unsafe fn create_swapchain(
    window: &Window,
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;
    let support = SwapchainSupport::get(instance, data, data.physical_device)?;

    let surface_format = get_swapchain_surface_format(&support.formats);
    let present_mode = get_swapchain_present_mode(&support.present_modes);
    let extent = get_swapchain_extent(window, support.capabilities);

    Ok(())
}
```

除去这些属性之外，我们还需要决定交换链中图像的数量。交换链有一个工作所需的最小图像数量：

```rust,noplaypen
let image_count = support.capabilities.min_image_count;
```

然而，仅仅满足这个最小值意味着我们有时候必须等待驱动程序完成内部操作，然后才能获取另一张图像来渲染。因此，建议至少请求比最小值多一张图像：

```rust,noplaypen
let image_count = support.capabilities.min_image_count + 1;
```

我们也要确保我们请求的图像数量不超过最大值，其中 `0` 是一个特殊值，表示没有最大值：

```rust,noplaypen
let mut image_count = support.capabilities.min_image_count + 1;
if support.capabilities.max_image_count != 0
    && image_count > support.capabilities.max_image_count
{
    image_count = support.capabilities.max_image_count;
}
```

接下来，我们需要说明如何处理在多个队列族中使用的交换链图像。如果图形队列族与呈现队列族不同，我们的应用程序就要在图形队列上绘制交换链中的图像，然后在呈现队列上提交它们。在这种情况下，我们需要指定如何处理在多个队列族中使用的交换链图像：

* `vk::SharingMode::EXCLUSIVE` &ndash; 一张图像同时只能被一个队列族持有，在另一个队列中使用它之前，必须显式地转移其所有权。这种方式能提供最好的性能。
* `vk::SharingMode::CONCURRENT` &ndash; 一张图像可以在多个队列族中使用，而不需要显式地转移所有权。

如果图形队列族和呈现队列族不同，我们的教程中会使用 `CONCURRENT` 模式，这样我们就不需要讲解所有权，毕竟这里面涉及的一些东西最好以后再详细解释。你必须使用 `queue_family_indices` 构建器方法提前指定哪些队列族之间可以共享交换链图像的所有权。如果图形队列族和呈现队列族相同 —— 大多数硬件都是这样的 —— 那么我们应该使用 `EXCLUSIVE` 模式，因为 `CONCURRENT` 模式要求你至少指定两个不同的队列族。

```rust,noplaypen
let mut queue_family_indices = vec![];
let image_sharing_mode = if indices.graphics != indices.present {
    queue_family_indices.push(indices.graphics);
    queue_family_indices.push(indices.present);
    vk::SharingMode::CONCURRENT
} else {
    vk::SharingMode::EXCLUSIVE
};
```

和其他的 Vulkan 对象一样，创建交换链对象也要填充一个巨大的结构体。又是熟悉的开始：

```rust,noplaypen
let info = vk::SwapchainCreateInfoKHR::builder()
    .surface(data.surface)
    // continued...
```

在指定交换链所绑定的表面之后，我们需要指定交换链图像的细节：

```rust,noplaypen
    .min_image_count(image_count)
    .image_format(surface_format.format)
    .image_color_space(surface_format.color_space)
    .image_extent(extent)
    .image_array_layers(1)
    .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
```

`image_array_layers` 指定每张图像的*层*（layer）数。除非你在开发一个立体 3D 应用程序，否则这个值总是 `1`。`image_usage` 位掩码指定我们会对交换链中的图像进行何种操作。在本教程中，我们将直接在图像上绘制，这意味着它们被用作颜色*附件*（color attachment）。先将图像渲染到另一个图像上、再执行后处理等操作也是可以的。在这种情况下，你可以使用 `vk::ImageUsageFlags::TRANSFER_DST` 这样的值，然后使用内存操作将渲染好的图像传输到交换链图像上。

```rust,noplaypen
    .image_sharing_mode(image_sharing_mode)
    .queue_family_indices(&queue_family_indices)
```

接着我们提供图像共享模式，以及允许共享交换链图像的队列族的索引。

```rust,noplaypen
    .pre_transform(support.capabilities.current_transform)
```

我们可以为交换链中的图像指定一个受支持的的变换操作（`capabilities` 的 `supported_transforms` 中记录了受支持的变换），例如 90 度顺时针旋转或水平翻转。如果你不想进行任何变换，只需指定当前变换 `current_transform` 即可。

```rust,noplaypen
    .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
```

`composite_alpha` 方法指定是否应该使用 alpha 通道与窗口系统中的其他窗口进行混合。你几乎总是希望忽略 alpha 通道，因此使用 `vk::CompositeAlphaFlagsKHR::OPAQUE`。

```rust,noplaypen
    .present_mode(present_mode)
    .clipped(true)
```

`present_mode` 的含义不言而喻。`clipped` 被设置为 `true` 来表示我们不关心被遮挡像素 —— 例如被窗口系统中其他窗口遮挡 —— 的颜色。除非你真的需要能够读取这些像素并获得可预测的结果，否则启用裁剪可以获得最佳性能。

```rust,noplaypen
    .old_swapchain(vk::SwapchainKHR::null());
```

还有最后一个方法，`old_swapchain`。你的交换链可能在应用程序运行时变得无效，或者不再是最优的 —— 例如当窗口大小改变的时候。在这种情况下，交换链实际上需要从头开始重建，而旧的交换链的引用必须在这个方法中指定。这是一个复杂的主题，我们将在以后的章节中讨论。现在我们假设我们只会创建一个交换链。我们可以省略这个调用，因为底层的字段默认就是一个空句柄，但为了完整起见，我们还是把它留在这里。

现在，向 `AppData` 中添加一个 `vk::SwapchainKHR` 字段来保存交换链对象：

```rust,noplaypen
struct AppData {
    // ...
    swapchain: vk::SwapchainKHR,
}
```

创建交换链就像调用 `create_swapchain_khr` 方法一样简单：

```rust,noplaypen
data.swapchain = device.create_swapchain_khr(&info, None)?;
```

不出所料，参数是交换链的创建信息和可选的自定义分配器。没有什么意外的。创建出的交换链需要在 `App::destroy` 中，在设备被销毁前清理掉：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_swapchain_khr(self.data.swapchain, None);
    // ...
}
```

现在运行程序，确保交换链创建成功。如果你在调用 `vkCreateSwapchainKHR` 的时候遇到了访问冲突错误，或者看到类似 `Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll` 的消息，那么请参考[常见问题](../faq.html)中关于 Steam 覆盖层的条目。

现在，不妨试试在校验层启用的情况下，在构造 `vk::SwapchainCreationInfoKHR` 结构体时去掉 `.image_extent(extent)` 这一行。你会发现，其中一个校验层立即就捕获到了错误，并打印出了一些有用的信息，指出 `image_extent` 的值非法：

![](../images/swapchain_validation_layer.png)

## 获取交换链图像

交换链已经创建出来了，现在我们还要获取交换链中的图像 `vk::Image` 的句柄。我们将在后面的章节中使用这些句柄来创建渲染目标。我们将在 `AppData` 中添加一个 `swapchain_images` 字段来保存这些句柄：

```rust,noplaypen
struct AppData {
    // ...
    swapchain_images: Vec<vk::Image>,
}
```

这些图像会随着交换链被创建出来，并且当交换链被销毁时被自动清理掉，因此我们不需要添加任何清理代码。

将下面的代码添加到 `create_swapchain` 函数的最后面，紧跟着 `create_swapchain_khr` 的调用，来获取这些句柄：

```rust,noplaypen
data.swapchain_images = device.get_swapchain_images_khr(data.swapchain)?;
```

还有一件事，我们需要保存交换链中图像的格式和交换范围，因为我们将在后面的章节中用到它们。在 `AppData` 中添加两个字段：

```rust,noplaypen
impl AppData {
    // ...
    swapchain_format: vk::Format,
    swapchain_extent: vk::Extent2D,
    swapchain: vk::SwapchainKHR,
    swapchain_images: Vec<vk::Image>,
}
```

然后在 `create_swapchain` 中保存它们：

```rust,noplaypen
data.swapchain_format = surface_format.format;
data.swapchain_extent = extent;
```

现在，我们有了一组可以绘制并呈现到屏幕上的图像。我们将在下一章中开始讨论如何将图像设置为渲染目标，然后开始使用图形管线和绘制指令来绘制图像！
