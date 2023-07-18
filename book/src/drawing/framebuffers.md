# 帧缓冲区

> 原文链接：<https://kylemayes.github.io/vulkanalia/drawing/framebuffers.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/13_framebuffers.rs)

我们在之前的几个章节中多次提到过帧缓冲区。而现在我们已经建立了渲染流程，就差一个跟交换链图片格式相同的帧缓冲区了。

<!-- TODO 无论如何，这一句读起来非常奇怪。之后需要参照 Vulkan Tutorial 原文反复确认。 -->
在创建渲染流程时指定的附件需要被包装进帧缓冲对象 `vk::Framebuffer` 中进行绑定。一个帧缓冲对象引用了所有代表附件的 `vk::ImageView` 对象。在我们的例子中，我们只有一个颜色附件。但这并不意味着我们就只需要使用一张图像，因为我们需要为交换链中的每个图像都创建对应的帧缓冲，并在渲染时使用与从交换链取得的图像对应的帧缓冲。

为此，我们在 `AppData` 中创建另一个 `Vec` 字段来存放帧缓冲区：

```rust,noplaypen
struct AppData {
    // ...
    framebuffers: Vec<vk::Framebuffer>,
}
```

我们创建一个新的函数 `create_framebuffers` ，在 `App::create` 里创建完管线之后调用它。之后我们将会在这个函数里创建 `vk::Framebuffer` 对象并将它们存储在 `framebuffers` 数组中：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_pipeline(&device, &mut data)?;
        create_framebuffers(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_framebuffers(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

在 `create_framebuffers` 函数中，遍历交换链中所有图像视图，为每个图像视图创建一个帧缓冲区：

```rust,noplaypen
unsafe fn create_framebuffers(device: &Device, data: &mut AppData) -> Result<()> {
    data.framebuffers = data
        .swapchain_image_views
        .iter()
        .map(|i| {
            let attachments = &[*i];
            let create_info = vk::FramebufferCreateInfo::builder()
                .render_pass(data.render_pass)
                .attachments(attachments)
                .width(data.swapchain_extent.width)
                .height(data.swapchain_extent.height)
                .layers(1);

            device.create_framebuffer(&create_info, None)
        })
        .collect::<Result<Vec<_>, _>>()?;

    Ok(())
}
```

如你所见，创建帧缓冲区的方式相当直白。首先我们需要指定帧缓冲要与哪个 `render_pass` 兼容。你只能在与一个帧缓冲兼容的渲染流程上使用它，这基本上就意味着渲染流程和帧缓冲有着相同数量和类型的附件。

`attachments` 字段指定了应绑定到渲染流程的 `attachment` 数组所对应的附件描述的 `vk::ImageView` 对象。

`width` 和 `height` 没什么好说的，就是宽度和高度，`layers` 则指的是图像的*层*数。我们交换链中的图像都是单个图像，所以层数为`1`。

我们应该在帧缓冲所基于的图像视图和渲染流程被销毁之前清理它：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.framebuffers
        .iter()
        .for_each(|f| self.device.destroy_framebuffer(*f, None));
    // ...
}
```

现在我们到达了一个里程碑，我们拥有了渲染所需的所有对象。在下一章中，我们将编写第一批实际的绘制指令。
