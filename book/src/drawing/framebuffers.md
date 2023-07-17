# Framebuffers 帧缓冲区

**Code:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/13_framebuffers.rs)

We've talked a lot about framebuffers in the past few chapters and we've set up the render pass to expect a single framebuffer with the same format as the swapchain images, but we haven't actually created any yet.

我们在过去的几个章节中谈过很多关于帧缓冲区的内容，我们已经设置了渲染通道，以期望一个与交换链图像格式相同的单一帧缓冲区，但我们还没有实际创建。

The attachments specified during render pass creation are bound by wrapping them into a `vk::Framebuffer` object. A framebuffer object references all of the `vk::ImageView` objects that represent the attachments. In our case that will be only a single one: the color attachment. However, the image that we have to use for the attachment depends on which image the swapchain returns when we retrieve one for presentation. That means that we have to create a framebuffer for all of the images in the swapchain and use the one that corresponds to the retrieved image at drawing time.

在创建渲染通道期间指定的附件通过将它们封装到`vk::Framebuffer`对象中进行绑定。帧缓冲区对象引用代表附件的所有`vk::ImageView`对象。在我们的情况下，将只有一个：颜色附件。然而，我们必须使用哪个图像作为附件取决于交换链在我们取回用于展示的图像时返回哪个图像。这意味着我们必须为交换链中的所有图像创建一个帧缓冲区，并在绘制时使用对应于取回图像的帧缓冲区。

To that end, create another `Vec` field in `AppData` to hold the framebuffers:

为此，在`AppData`中创建另一个`Vec`字段来存放帧缓冲区：

```rust,noplaypen
struct AppData {
    // ...
    framebuffers: Vec<vk::Framebuffer>,
}
```

We'll create the objects for this array in a new function `create_framebuffers` that is called from `App::create` right after creating the graphics pipeline:

我们将在一个新的函数`create_framebuffers`中为这个数组创建对象，该函数在创建图形管道之后从`App::create`中调用：

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

Start by mapping over the swapchain image views:

首先映射交换链图像视图：

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

We'll then create a framebuffer for each image view:

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

As you can see, creation of framebuffers is quite straightforward. We first need to specify with which `render_pass` the framebuffer needs to be compatible. You can only use a framebuffer with the render passes that it is compatible with, which roughly means that they use the same number and type of attachments.

如你所见，创建帧缓冲区相当直观。我们首先需要明确帧缓冲区需要与哪个`render_pass`兼容。你只能使用与其兼容的渲染通道的帧缓冲区，大致上这意味着它们使用相同数量和类型的附件。

The `attachments` field specifies the `vk::ImageView` objects that should be bound to the respective attachment descriptions in the render pass `attachment` array.

`attachments`字段指定应绑定到渲染通道`attachment`数组中的相应附件描述的`vk::ImageView`对象。

The `width` and `height` parameters are self-explanatory and `layers` refers to the number of layers in image arrays. Our swapchain images are single images, so the number of layers is `1`.

`width`和`height`参数自我解释，`layers`指的是图像数组中的层数。我们的交换链图像都是单个图像，所以层数为`1`。

We should delete the framebuffers before the image views and render pass that they are based on, but only after we've finished rendering:

我们应该在帧缓冲区所基于的图像视图和渲染通道被删除之前删除帧缓冲区，但只有在我们完成渲染后：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.framebuffers
        .iter()
        .for_each(|f| self.device.destroy_framebuffer(*f, None));
    // ...
}
```

We've now reached the milestone where we have all of the objects that are required for rendering. In the next chapter we're going to write the first actual drawing commands.

我们现在已经到达了一个里程碑，我们拥有了所有用于渲染所需的对象。在下一章中，我们将编写第一批实际的绘图命令。