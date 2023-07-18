# 帧缓冲区

**代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/13_framebuffers.rs)

在过去的几个章节里我们多次提到了帧缓冲区，我们现在已经建立了渲染环节，就差一个跟交换链图片格式相同的帧缓冲区了，但我们还没有实际创建过。

在创建渲染环节的时候指定的附件是通过封装到 `vk::Framebuffer` 对象里进行绑定的。帧缓冲区对象引用着所有附件代表的 `vk::ImageView` 对象。对我们来说就只有一个：颜色附件。然而，我们能用于附件的图像取决于从交换链中取得的用于呈现的是哪张。这意味着我们必须为交换链中每个图像都创建一个帧缓冲区，并在绘制时使用取得的图像所对应的那个帧缓冲区。

为此，在 `AppData` 中创建另一个 `Vec` 字段来存放帧缓冲区：

```rust,noplaypen
struct AppData {
    // ...
    framebuffers: Vec<vk::Framebuffer>,
}
```

我们创建一个新的函数 `create_framebuffers` ，在 `App::create` 里创建完管线之后调用它。之后我们将会在这个函数里为这个数组创建对象。

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

首先对交换链图像视图进行映射：

```rust,noplaypen
unsafe fn create_framebuffers(device: &Device, data: &mut AppData) -> Result<()> {
    data.framebuffers = data
        .swapchain_image_views
        .iter()
        .map(|i| {

        })
        .collect::<Result<Vec<_>, _>>()?;

    Ok(())
}
```

然后为每个图像视图创建一个帧缓冲区：

```rust,noplaypen
let attachments = &[*i];
let create_info = vk::FramebufferCreateInfo::builder()
    .render_pass(data.render_pass)
    .attachments(attachments)
    .width(data.swapchain_extent.width)
    .height(data.swapchain_extent.height)
    .layers(1);

device.create_framebuffer(&create_info, None)
```

如你所见，创建帧缓冲区的方式相当直白。我们首先需要明确帧缓冲区需要与哪个 `render_pass` 兼容。你只能在与一个帧缓存区兼容的渲染环节上使用它，这基本上就意味着它们有着相同数量和类型的附件。

`attachments` 字段指定了应绑定到渲染环节的 `attachment` 数组所对应的附件描述的 `vk::ImageView` 对象。

`width` 和 `height` 没什么好说的，就是宽度和高度，`layers` 则指的是图像数组中的层数。我们的交换链图像都是单个图像，所以层数为`1`。

我们应在帧缓冲区所基于的图像视图和渲染环节被删除之前删除它，不过得等我们完成渲染后：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.framebuffers
        .iter()
        .for_each(|f| self.device.destroy_framebuffer(*f, None));
    // ...
}
```

现在我们到达了一个里程碑，我们拥有了所有渲染所必需的对象。在下一章中，我们将编写第一批实际的绘制指令。