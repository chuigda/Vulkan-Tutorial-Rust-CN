# 多重采样

> 原文链接：<https://kylemayes.github.io/vulkanalia/quality/multisampling.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/29_multisampling.rs)

现在我们的程序为纹理加载了多个细节层级，这能修复渲染远离观察者的物体时的伪影。图像现在更加平滑了，但是如果仔细观察，你会发现在几何形状的边缘有锯齿状的图案。这在我们早期的程序中就能看到，当我们渲染一个四边形时：

![](../images/texcoord_visualization.png)

这种我们不希望看到的效果被称为“锯齿”（aliasing），它是由于渲染可用的像素数量有限造成的。因为没有显示器的分辨率是无限的，所以锯齿总是会在某种程度上可见。有很多方法可以修复这个问题，在本章中我们将会专注于其中一个比较流行的方法：[多重采样抗锯齿](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)（multisample anti-aliasing, MSAA）。

在通常的渲染中，像素的颜色是由单个样本点决定的，这个样本点在大多数情况下是屏幕上目标像素的中心。而如果绘制的线段通过某个像素，但是没有覆盖样本点，那么这个像素就会被留空，导致了锯齿状的“楼梯”效果。

![](../images/aliasing.png)

MSAA 所做的就是使用多个样本点来决定像素的最终颜色（因此得名）。正如你所期望的那样，更多的样本会得到更好的结果，但也会消耗更多计算资源。

![](../images/antialiasing.png)

在我们的实现中，我们会使用可用的最大样本数量。取决于你的应用程序，这可能不是最好的方法。如果更少的样本就能满足你的质量要求，那么为了性能，使用更少的样本可能会更好。

## 获取可用的样本数量

Let's start off by determining how many samples our hardware can use. Most modern GPUs support at least 8 samples but this number is not guaranteed to be the same everywhere. We'll keep track of it by adding a new field to `AppData`:

让我们从检测硬件支持多少样本开始。大多数现代 GPU 至少支持 8 个样本，但这个数字不保证在任何地方都是一样的。我们在 `AppData` 中添加一个新字段来记录它：

```rust,noplaypen
struct AppData {
    // ...
    physical_device: vk::PhysicalDevice,
    msaa_samples: vk::SampleCountFlags,
    // ...
}
```

默认情况下，我们为每个像素使用一个样本，相当于没有多重采样，这种情况下最终图像将保持不变。最大样本数可以从与我们选择的物理设备相关联的 `vk::PhysicalDeviceProperties` 中提取。我们使用了深度缓冲，因此我们必须考虑将颜色和深度的样本数都考虑在内。两者都支持的最高样本数（使用 `&` 运算符）将是我们可以支持的最大样本数。添加一个函数来获取这些信息：

```rust,noplaypen
unsafe fn get_max_msaa_samples(
    instance: &Instance,
    data: &AppData,
) -> vk::SampleCountFlags {
    let properties = instance.get_physical_device_properties(data.physical_device);
    let counts = properties.limits.framebuffer_color_sample_counts
        & properties.limits.framebuffer_depth_sample_counts;
    [
        vk::SampleCountFlags::_64,
        vk::SampleCountFlags::_32,
        vk::SampleCountFlags::_16,
        vk::SampleCountFlags::_8,
        vk::SampleCountFlags::_4,
        vk::SampleCountFlags::_2,
    ]
    .iter()
    .cloned()
    .find(|c| counts.contains(*c))
    .unwrap_or(vk::SampleCountFlags::_1)
}
```

现在我们在选取物理设备的环节使用这个函数来设置 `msaa_samples` 变量。我们稍微修改 `pick_physical_device` 函数，在选取物理设备之后设置最大的 MSAA 样本数：

```rust,noplaypen
unsafe fn pick_physical_device(instance: &Instance, data: &mut AppData) -> Result<()> {
    // ...

    for physical_device in instance.enumerate_physical_devices()? {
        // ...

        if let Err(error) = check_physical_device(instance, data, physical_device) {
            // ...
        } else {
            // ...
            data.msaa_samples = get_max_msaa_samples(instance, data);
            return Ok(());
        }
    }

    Ok(())
}
```

## 设置渲染目标

<!-- 和这里 https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/pull/74#discussion_r1341886973 一样，真的只有一个绘制操作处于活动状态吗？ -->
在 MSAA 中，每个像素都在一个离屏缓冲中进行采样，然后离屏缓冲被渲染到屏幕上。这个新的缓冲与我们一直渲染到的普通图像略有不同 —— 它们必须能为每个像素存储多个样本。一旦创建了多重采样缓冲，就必须将其解析到默认的帧缓冲（每个像素只存储一个样本）。这就是为什么我们必须创建一个额外的渲染目标并修改我们当前的绘制过程。我们只需要一个渲染目标，因为一次只能有一个绘制操作处于活动状态，就像深度缓冲一样。在 `AppData` 中添加以下字段：

```rust,noplaypen
struct AppData {
    // ...
    color_image: vk::Image,
    color_image_memory: vk::DeviceMemory,
    color_image_view: vk::ImageView,
    // ...
}
```

This new image will have to store the desired number of samples per pixel, so we need to pass this number to `vk::ImageCreateInfo` during the image creation process. Modify the `^create_image` function by adding a `samples` parameter:

这个新的图像需要为每个像素存储所需数量的样本，因此我们需要在创建图像的过程中将样本数传递给 `vk::ImageCreateInfo`。修改 `create_image` 函数，添加一个 `samples` 参数：

```rust,noplaypen
unsafe fn create_image(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    width: u32,
    height: u32,
    mip_levels: u32,
    samples: vk::SampleCountFlags,
    format: vk::Format,
    tiling: vk::ImageTiling,
    usage: vk::ImageUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> Result<(vk::Image, vk::DeviceMemory)> {
    // Image

    let info = vk::ImageCreateInfo::builder()
        // ...
        .samples(samples)
        // ...

    // ...
}
```

现在我们先暂时给所有调用这个函数的地方传递 `vk::SampleCountFlags::_1`，我们将在实现过程中逐步将其替换为正确的值：

```rust,noplaypen
let (depth_image, depth_image_memory) = create_image(
    instance,
    device,
    data,
    data.swapchain_extent.width,
    data.swapchain_extent.height,
    1,
    vk::SampleCountFlags::_1,
    format,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
)?;

// ...

let (texture_image, texture_image_memory) = create_image(
    instance,
    device,
    data,
    width,
    height,
    data.mip_levels,
    vk::SampleCountFlags::_1,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::SAMPLED
        | vk::ImageUsageFlags::TRANSFER_DST
        | vk::ImageUsageFlags::TRANSFER_SRC,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
)?;
```

现在我们将创建一个多重采样的颜色缓冲。添加一个 `create_color_objects` 函数，并注意我们在这里将 `data.msaa_samples` 传递给 `create_image`。我们只使用一个多级渐远层级，因为 Vulkan 规范要求每个像素有多个样本的图像必须只有一个多级渐远层级。此外，这个颜色缓冲不需要多级渐远层级，因为它不会被用作纹理：

```rust,noplaypen
unsafe fn create_color_objects(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let (color_image, color_image_memory) = create_image(
        instance,
        device,
        data,
        data.swapchain_extent.width,
        data.swapchain_extent.height,
        1,
        data.msaa_samples,
        data.swapchain_format,
        vk::ImageTiling::OPTIMAL,
        vk::ImageUsageFlags::COLOR_ATTACHMENT
            | vk::ImageUsageFlags::TRANSIENT_ATTACHMENT,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    )?;

    data.color_image = color_image;
    data.color_image_memory = color_image_memory;

    data.color_image_view = create_image_view(
        device,
        data.color_image,
        data.swapchain_format,
        vk::ImageAspectFlags::COLOR,
        1,
    )?;

    Ok(())
}
```

为了保持一致性，在 `create_depth_objects` 之前调用这个函数：

```rust,noplaypen
unsafe fn create(window: &Window) -> Result<Self> {
    // ...
    create_color_objects(&instance, &device, &mut data)?;
    create_depth_objects(&instance, &device, &mut data)?;
    // ...
}
```

现在我们已经有了一个多重采样的颜色缓冲，是时候来处理深度了。修改 `create_depth_objects`，更新深度缓冲使用的样本数：

```rust,noplaypen
unsafe fn create_depth_objects(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    // ...

    let (depth_image, depth_image_memory) = create_image(
        instance,
        device,
        data,
        data.swapchain_extent.width,
        data.swapchain_extent.height,
        1,
        data.msaa_samples,
        format,
        vk::ImageTiling::OPTIMAL,
        vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    )?;

    // ...
}
```

我们现在创建了一些新的 Vulkan 资源，所以不要忘记在必要时释放它们：

```rust,noplaypen
unsafe fn destroy_swapchain(&mut self) {
    self.device.destroy_image_view(self.data.color_image_view, None);
    self.device.free_memory(self.data.color_image_memory, None);
    self.device.destroy_image(self.data.color_image, None);
    // ...
}
```

然后更新 `App::recreate_swapchain` 方法，这样当窗口大小改变时，新的颜色图像就能以正确的分辨率重新创建：

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    // ...
    create_color_objects(&self.instance, &self.device, &mut self.data)?;
    create_depth_objects(&self.instance, &self.device, &mut self.data)?;
    // ...
}
```

我们已经完成了初始的 MSAA 设置，现在我们需要在图形管线、帧缓冲和渲染流程中开始使用这个新资源，然后看看效果！

## 添加新的附件

首先处理渲染流程。修改 `create_render_pass`，更新颜色和深度附件创建信息：

```rust,noplaypen
unsafe fn create_render_pass(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let color_attachment = vk::AttachmentDescription::builder()
        // ...
        .samples(data.msaa_samples)
        // ...
        .final_layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL);

    let depth_stencil_attachment = vk::AttachmentDescription::builder()
        // ...
        .samples(data.msaa_samples)
        // ...

    // ...
}
```

你会注意到我们已经将 `final_layout` 从 `vk::ImageLayout::PRESENT_SRC_KHR` 改为了 `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL`。这是因为多重采样图像不能直接呈现。我们首先需要将它们解析为普通图像。这个要求不适用于深度缓冲，因为它不会在任何时候被呈现。因此，我们只需要为颜色添加一个新的附件，这是一个所谓的解析附件：

```rust,noplaypen
let color_resolve_attachment = vk::AttachmentDescription::builder()
    .format(data.swapchain_format)
    .samples(vk::SampleCountFlags::_1)
    .load_op(vk::AttachmentLoadOp::DONT_CARE)
    .store_op(vk::AttachmentStoreOp::STORE)
    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
    .initial_layout(vk::ImageLayout::UNDEFINED)
    .final_layout(vk::ImageLayout::PRESENT_SRC_KHR);
```

现在必须告诉渲染流程将多重采样的颜色图像解析为普通附件。创建一个新的附件引用，它将指向颜色缓冲，该缓冲将用作解析目标：

```rust,noplaypen
let color_resolve_attachment_ref = vk::AttachmentReference::builder()
    .attachment(2)
    .layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL);
```

让 `resolve_attachments` 成员指向新创建的附件引用。这就足以让渲染流程定义一个能将图像渲染到屏幕上的多重采样解析操作了：

```rust,noplaypen
let color_attachments = &[color_attachment_ref];
let resolve_attachments = &[color_resolve_attachment_ref];
let subpass = vk::SubpassDescription::builder()
    .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
    .color_attachments(color_attachments)
    .depth_stencil_attachment(&depth_stencil_attachment_ref)
    .resolve_attachments(resolve_attachments);
```

现在用新的颜色附件更新渲染流程创建信息：

```rust,noplaypen
let attachments = &[
    color_attachment,
    depth_stencil_attachment,
    color_resolve_attachment,
];
let subpasses = &[subpass];
let dependencies = &[dependency];
let info = vk::RenderPassCreateInfo::builder()
    .attachments(attachments)
    .subpasses(subpasses)
    .dependencies(dependencies);
```

渲染流程就位了，修改 `create_framebuffers`，将新的图像视图添加到附件切片中：

```rust,noplaypen
let attachments = &[data.color_image_view, data.depth_image_view, *i];
```

最后，修改 `create_pipeline` 来告诉新创建的管线使用多个样本：

```rust,noplaypen
let multisample_state = vk::PipelineMultisampleStateCreateInfo::builder()
    .sample_shading_enable(false)
    .rasterization_samples(data.msaa_samples);
```

现在运行程序，你应该会看到下面的效果：

![](../images/multisampling.png)

就像多级渐远一样，差异可能不会立即显现出来。仔细观察，你会注意到边缘不再那么锯齿状，整个图像与原始图像相比似乎更加平滑（如果你在单独的标签页中打开下面的图像，差异会更容易被发现）。

![](../images/multisampling_comparison.png)

再把斧头头部放大 8 倍，差异更加明显：

![](../images/multisampling_comparison_axe.png)

## 质量提升

我们目前的 MSAA 实现有一些局限性，在细节更多的场景中，这可能影响输出画面的质量。例如，现在我们还没有解决着色器锯齿会造成的潜在问题，也就是说，MSAA 只平滑了图形的边缘，但却没有处理内部的填充。这可能会导致这样一种情况：屏幕上渲染出了一个平滑的多边形，但是如果应用的纹理包含高对比度的颜色，那么纹理看起来仍然是锯齿状的。解决这个问题的一种方法是启用[采样着色](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#primsrast-sampleshading)，这将进一步提高图像质量，但会带来额外的性能开销：

```rust,noplaypen
unsafe fn create_logical_device(
    instance: &Instance,
    data: &mut AppData,
) -> Result<Device> {
    // ...

    let features = vk::PhysicalDeviceFeatures::builder()
        .sampler_anisotropy(true)
        // Enable sample shading feature for the device.
        .sample_rate_shading(true);

    // ...
}

// ...

unsafe fn create_pipeline(device: &Device, data: &mut AppData) -> Result<()> {
    // ...

    let multisample_state = vk::PipelineMultisampleStateCreateInfo::builder()
        // Enable sample shading in the pipeline.
        .sample_shading_enable(true)
        // Minimum fraction for sample shading; closer to one is smoother.
        .min_sample_shading(0.2)
        .rasterization_samples(data.msaa_samples);

    // ...
}
```

在这个例子中，我们将采样着色保持禁用。但在某些情况下，采样着色可以带来显著的质量提升：

![](../images/sample_shading.png)
