# 交换链

> 原文链接：<https://kylemayes.github.io/vulkanalia/presentation/swapchain.html>
>
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/06_swapchain_creation.rs)

Vulkan 没有“默认帧缓冲”（default framebuffer）的概念，因此，Vulkan 需要一个架构来持有我们将要绘制的帧缓冲。这个架构就是*交换链*，在 Vulkan 中它必须被显式创建。交换链本质上就是一个队列，其中充满了等待呈现到屏幕上的图像。我们的应用程序每次会从这个队列中获取一张图像，在上面绘制，然后将它返还到队列中。队列如何工作以及何时呈现队列中的图像取决于交换链的设置，但通常来说，交换链的目的是使图像的呈现与屏幕刷新率同步。

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

Using a swapchain requires enabling the `VK_KHR_swapchain` extension first. Enabling the extension just requires a small change to our list of device extensions in the `create_logical_device` function. Initialize our list of device extensions with a list of null-terminated strings constructed from `DEVICE_EXTENSIONS`:

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
* 交换范围（交换链中图像的分辨率）

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
* `vk::PresentModeKHR::MAILBOX` &ndash; 这是 `FIFO` 模式的另一种变体。如果程序提交图像的速度比显示器刷新的速度快，当队列已满时，队列中的图像会直接被新的图像替换，而不会阻塞应用程序。这种模式可以用来尽可能快地渲染帧，同时避免撕裂，因而比标准的垂直同步有更少的延迟。这通常被称为“三重缓冲”，尽管仅靠三个缓冲区本身并不能让帧率不受限制。

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

<!-- TODO need refinement -->
交换范围就是交换链中图像的分辨率，它几乎总是等于我们正在绘制的窗口的分辨率。可用的分辨率范围在 `vk::SurfaceCapabilitiesKHR` 结构体中定义。Vulkan 告诉我们要匹配窗口的分辨率，方法是在 `current_extent` 成员中设置宽度和高度。然而，有些窗口管理器允许我们在这里做一些改变，这是通过将 `current_extent` 中的宽度和高度设置为一个特殊值来指示的：`u32` 的最大值。在这种情况下，我们将选择最佳匹配窗口的分辨率，这个分辨率必须在 `min_image_extent` 和 `max_image_extent` 的范围内。

The swap extent is the resolution of the swapchain images and it's almost always exactly equal to the resolution of the window that we're drawing to. The range of the possible resolutions is defined in the `vk::SurfaceCapabilitiesKHR` structure. Vulkan tells us to match the resolution of the window by setting the width and height in the `current_extent` member. However, some window managers do allow us to differ here and this is indicated by setting the width and height in `current_extent` to a special value: the maximum value of `u32`. In that case we'll pick the resolution that best matches the window within the `min_image_extent` and `max_image_extent` bounds.

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

We define the `clamp` function to restrict the actual size of the window within the supported range supported by the Vulkan device.

## Creating the swapchain

Now that we have all of these helper functions assisting us with the choices we have to make at runtime, we finally have all the information that is needed to create a working swapchain.

Create a `create_swapchain` function that starts out with the results of these calls and make sure to call it from `App::create` after logical device creation.

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

Aside from these properties we also have to decide how many images we would like to have in the swapchain. The implementation specifies the minimum number that it requires to function:

```rust,noplaypen
let image_count = support.capabilities.min_image_count;
```

However, simply sticking to this minimum means that we may sometimes have to wait on the driver to complete internal operations before we can acquire another image to render to. Therefore it is recommended to request at least one more image than the minimum:

```rust,noplaypen
let image_count = support.capabilities.min_image_count + 1;
```

We should also make sure to not exceed the maximum number of images while doing this, where `0` is a special value that means that there is no maximum:

```rust,noplaypen
let mut image_count = support.capabilities.min_image_count + 1;
if support.capabilities.max_image_count != 0
    && image_count > support.capabilities.max_image_count
{
    image_count = support.capabilities.max_image_count;
}
```

Next, we need to specify how to handle swapchain images that will be used across multiple queue families. That will be the case in our application if the graphics queue family is different from the presentation queue. We'll be drawing on the images in the swapchain from the graphics queue and then submitting them on the presentation queue. There are two ways to handle images that are accessed from multiple queues:

* `vk::SharingMode::EXCLUSIVE` &ndash; An image is owned by one queue family at a time and ownership must be explicitly transferred before using it in another queue family. This option offers the best performance.
* `vk::SharingMode::CONCURRENT` &ndash; Images can be used across multiple queue families without explicit ownership transfers.

If the queue families differ, then we'll be using the concurrent mode in this tutorial to avoid having to do the ownership chapters, because these involve some concepts that are better explained at a later time. Concurrent mode requires you to specify in advance between which queue families ownership will be shared using the `queue_family_indices` builder method If the graphics queue family and presentation queue family are the same, which will be the case on most hardware, then we should stick to exclusive mode, because concurrent mode requires you to specify at least two distinct queue families.

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

As is tradition with Vulkan objects, creating the swapchain object requires filling in a large structure. It starts out very familiarly:

```rust,noplaypen
let info = vk::SwapchainCreateInfoKHR::builder()
    .surface(data.surface)
    // continued...
```

After specifying which surface the swapchain should be tied to, the details of the swapchain images are specified:

```rust,noplaypen
    .min_image_count(image_count)
    .image_format(surface_format.format)
    .image_color_space(surface_format.color_space)
    .image_extent(extent)
    .image_array_layers(1)
    .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
```

The `image_array_layers` specifies the amount of layers each image consists of. This is always `1` unless you are developing a stereoscopic 3D application. The `image_usage` bitmask specifies what kind of operations we'll use the images in the swapchain for. In this tutorial we're going to render directly to them, which means that they're used as color attachment. It is also possible that you'll render images to a separate image first to perform operations like post-processing. In that case you may use a value like `vk::ImageUsageFlags::TRANSFER_DST` instead and use a memory operation to transfer the rendered image to a swapchain image.

```rust,noplaypen
    .image_sharing_mode(image_sharing_mode)
    .queue_family_indices(&queue_family_indices)
```

Next we'll provide the image sharing mode and indices of the queue families permitted to share the swapchain images.

```rust,noplaypen
    .pre_transform(support.capabilities.current_transform)
```

We can specify that a certain transform should be applied to images in the swapchain if it is supported (`supported_transforms` in `capabilities`), like a 90 degree clockwise rotation or horizontal flip. To specify that you do not want any transformation, simply specify the current transformation.

```rust,noplaypen
    .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
```

The `composite_alpha` method specifies if the alpha channel should be used for blending with other windows in the window system. You'll almost always want to simply ignore the alpha channel, hence `vk::CompositeAlphaFlagsKHR::OPAQUE`.

```rust,noplaypen
    .present_mode(present_mode)
    .clipped(true)
```

The `present_mode` member speaks for itself. If the `clipped` member is set to `true` then that means that we don't care about the color of pixels that are obscured, for example because another window is in front of them. Unless you really need to be able to read these pixels back and get predictable results, you'll get the best performance by enabling clipping.

```rust,noplaypen
    .old_swapchain(vk::SwapchainKHR::null());
```

That leaves one last method, `old_swapchain`. With Vulkan it's possible that your swapchain becomes invalid or unoptimized while your application is running, for example because the window was resized. In that case the swapchain actually needs to be recreated from scratch and a reference to the old one must be specified in this method. This is a complex topic that we'll learn more about in a future chapter. For now we'll assume that we'll only ever create one swapchain. We could omit this method since the underlying field will default to a null handle, but we'll leave it in for completeness.

Now add an `AppData` field to store the `vk::SwapchainKHR` object:

```rust,noplaypen
struct AppData {
    // ...
    swapchain: vk::SwapchainKHR,
}
```

Creating the swapchain is now as simple as calling `create_swapchain_khr`:

```rust,noplaypen
data.swapchain = device.create_swapchain_khr(&info, None)?;
```

The parameters are the swapchain creation info and optional custom allocators. No surprises there. It should be cleaned up in `App::destroy` before the device:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_swapchain_khr(self.data.swapchain, None);
    // ...
}
```

Now run the application to ensure that the swapchain is created successfully! If at this point you get an access violation error in `vkCreateSwapchainKHR` or see a message like `Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll`, then see the [FAQ entry](../faq.html) about the Steam overlay layer.

Try removing the `.image_extent(extent)` line from where you are building the `vk::SwapchainCreateInfoKHR` struct with validation layers enabled. You'll see that one of the validation layers immediately catches the mistake and some helpful messages are printed which call out the illegal value provided for `image_extent`:

![](../images/swapchain_validation_layer.png)

## Retrieving the swapchain images

The swapchain has been created now, so all that remains is retrieving the handles of the `vk::Image`s in it. We'll reference these during rendering operations in later chapters. Add an `AppData` field to store the handles:

```rust,noplaypen
struct AppData {
    // ...
    swapchain_images: Vec<vk::Image>,
}
```

The images were created by the implementation for the swapchain and they will be automatically cleaned up once the swapchain has been destroyed, therefore we don't need to add any cleanup code.

I'm adding the code to retrieve the handles to the end of the `create_swapchain` function, right after the `create_swapchain_khr` call.

```rust,noplaypen
data.swapchain_images = device.get_swapchain_images_khr(data.swapchain)?;
```

One last thing, store the format and extent we've chosen for the swapchain images in `AppData` fields. We'll need them in future chapters.

```rust,noplaypen
impl AppData {
    // ...
    swapchain_format: vk::Format,
    swapchain_extent: vk::Extent2D,
    swapchain: vk::SwapchainKHR,
    swapchain_images: Vec<vk::Image>,
}
```

And then in `create_swapchain`:

```rust,noplaypen
data.swapchain_format = surface_format.format;
data.swapchain_extent = extent;
```

We now have a set of images that can be drawn onto and can be presented to the window. The next chapter will begin to cover how we can set up the images as render targets and then we start looking into the actual graphics pipeline and drawing commands!
