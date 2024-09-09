# 渲染流程

> 原文链接：<https://kylemayes.github.io/vulkanalia/pipeline/render_passes.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

**本章代码** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/11_render_passes.rs)

在创建渲染管线之前，我们还需要设置渲染过程中将会使用的帧缓冲附件（framebuffer attachments）。我们需要指定有多少个颜色缓冲和深度缓冲，每个缓冲使用多少样本数，以及渲染操作将如何处理缓冲中的内容。所有这些信息都会被装进一个*渲染流程*对象中。我们将会创建一个新的函数 `create_render_pass`，并在 `App::create` 中 `create_pipeline` 之前调用它：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_render_pass(&instance, &device, &mut data)?;
        create_pipeline(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_render_pass(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

## 附件描述

在我们的场景中，我们只需要一个颜色附件。我们会在 `create_render_pass` 函数中创建一个 `vk::AttachmentDescription` 来表示它：

```rust,noplaypen
let color_attachment = vk::AttachmentDescription::builder()
    .format(data.swapchain_format)
    .samples(vk::SampleCountFlags::_1)
    // continued...
```

颜色附件的 `format` 字段需要与交换链图像的格式匹配。我们现在不会进行多重采样，所以我们只用 1 个样本。

```rust,noplaypen
    .load_op(vk::AttachmentLoadOp::CLEAR)
    .store_op(vk::AttachmentStoreOp::STORE)
```

`load_op` 和 `store_op` 决定在渲染之前和之后对附件中的数据做什么。对于 `load_op` 我们有以下三种选择：

* `vk::AttachmentLoadOp::LOAD` &ndash; 保留附件中已有的内容
* `vk::AttachmentLoadOp::CLEAR` &ndash; 在渲染开始前将附件清空，为每个像素设置一个常量值
* `vk::AttachmentLoadOp::DONT_CARE` &ndash; 附件中已有的内容是未定义的，我们不关心它们

在我们的场景下，我们希望在渲染新帧之前将帧缓冲清空为黑色。而 `store_op` 只有两种选择：

* `vk::AttachmentStoreOp::STORE` &ndash; 渲染的内容将会被存储起来，以便之后读取
* `vk::AttachmentStoreOp::DONT_CARE` &ndash; 渲染结束后帧缓冲中的内容是未定义的

我们希望在屏幕上看到渲染出来的三角形，所以我们选择 `STORE`。

```rust,noplaypen
    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
```

`load_op` 和 `store_op` 对颜色和深度数据生效，而 `stencil_load_op` 和 `stencil_store_op` 对模板数据生效。我们的应用不会使用模板缓冲，所以加载和存储的结果并不重要。

```rust,noplaypen
    .initial_layout(vk::ImageLayout::UNDEFINED)
    .final_layout(vk::ImageLayout::PRESENT_SRC_KHR);
```

在 Vulkan 中，纹理和帧缓冲是以具有特定像素格式的 `vk::Image` 对象来表示的。不过你可以根据你在对图像做的事情改变内存中像素的布局。

一些常见的布局包括：

* `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL` &ndash; 图像会被用作颜色附件
* `vk::ImageLayout::PRESENT_SRC_KHR` &ndash; 图像被用在交换链中进行呈现操作
* `vk::ImageLayout::TRANSFER_DST_OPTIMAL` &ndash; 图像用作赋值操作的目标

我们会在纹理章节中对这一主题进行更深入的探讨，不过现在我们只要知道图像需要先被转换到特定的布局，以便进行下一步的操作。

`initial_layout` 指定图像在渲染流程开始前所具有的布局，而 `final_layout` 指定图像在渲染流程结束后将会自动转换到的布局。我们将 `initial_layout` 设置为 `vk::ImageLaout::UNDEFINED`，表明我们不关心图像输入时的布局。这也意味着图像中的内容不一定会被保留，但没关系，反正我们也打算清空它了。而在渲染之后，我们希望图像可以被在交换链上呈现，所以我们将 `final_layout` 设置为 `vk::ImageLayout::PRESENT_SRC_KHR`。

## 子流程与附件引用

一个渲染流程可以由多个子流程组成。子流程是一系列的渲染操作，每个渲染操作都依赖于之前的子流程处理后帧缓冲中的内容。例如，许多后处理（post-processing）效果就是前面的处理结果上叠加一系列操作来实现的。如果你将这些渲染操作组合成一个渲染流程，Vulkan 就可以对这些操作进行重新排序，以便更好地利用内存带宽来提高性能。不过在我们的第一个三角形程序中，我们只需要一个子流程。

每个子流程都要引用一个或多个附件，这些附件通过 `vk::AttachmentReference` 结构体指定：

```rust,noplaypen
let color_attachment_ref = vk::AttachmentReference::builder()
    .attachment(0)
    .layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL);
```

`attachment` 参数通过附件描述符数组的索引指定要引用哪个附件。我们的附件描述数组中只有一个 `vk::AttachmentDescription`，所以我们将索引置为 `0`。`layout` 用于指定子流程开始时附件的布局，Vulkan 会在子流程开始时自动将附件转换到这个布局。我们打算将附件用作颜色缓冲，所以 `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL` 布局会给我们最好的性能，正如它的名字所说的那样。

子流程使用 `vk::SubpassDescription` 结构体来描述：

```rust,noplaypen
let color_attachments = &[color_attachment_ref];
let subpass = vk::SubpassDescription::builder()
    .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
    .color_attachments(color_attachments);
```

Vulkan 在将来也可能支持计算子流程，所以我们需要明确指定这是一个图形子流程。然后我们指定对颜色附件的引用。

这里设置的颜色附着将会被片元着色器使用，对应我们在片元着色器中的 `layout(location = 0) out vec4 outColor` 指令。

以下类型的附件也可以被子流程使用：

* `input_attachments` &ndash; 可以被着色器读取的附件
* `resolve_attachments` &ndash; 用于多重采样颜色附件的附件
* `depth_stencil_attachment` &ndash; 用于深度和模板数据的附件
* `preserve_attachments` &ndash; 子流程不会使用的附件，但其中的数据必须被保留

## 渲染流程

现在，我们已经设置好了附件和与之关联的子流程，我们可以开始创建渲染流程了。在 `AppData` 中 `pipeline_layout` 字段的上面添加一个成员变量来存储 `vk::RenderPass` 对象：

```rust,noplaypen
struct AppData {
    // ...
    render_pass: vk::RenderPass,
    pipeline_layout: vk::PipelineLayout,
}
```

接着，我们就可以用附件和子流程数组填充 `vk::RenderPassCreateInfo` 结构体，来创建渲染渲染对象了：

```rust,noplaypen
let attachments = &[color_attachment];
let subpasses = &[subpass];
let info = vk::RenderPassCreateInfo::builder()
    .attachments(attachments)
    .subpasses(subpasses);

data.render_pass = device.create_render_pass(&info, None)?;
```

和管线布局一样，渲染流程会在整个程序中被使用，所以我们只在 `App::destroy` 中清理它：

Just like the pipeline layout, the render pass will be referenced throughout the program, so it should only be cleaned up at the end in `App::destroy`:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_pipeline_layout(self.data.pipeline_layout, None);
    self.device.destroy_render_pass(self.data.render_pass, None);
    // ...
}
```

到此为止我们已经做了很多工作，在下一章中，我们会将这些工作整合起来，最终创建出图形管线对象！
