# 图像视图 (Image views)

> 原文链接：<https://kylemayes.github.io/vulkanalia/presentation/image_views.html>
>
> Commit Hash: ceb4a3fc6d8ca565af4f8679c4889bcad7941338

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/07_image_views.rs)

要在渲染管线中使用任何 `vk::Image` —— 包括交换链中的那些，我们都需要为其创建一个图像视图对象 `vk::ImageView`。图像视图就像它的名字所描述的那样，它描述了如何访问图像，以及访问图像的哪一部分。例如，图像视图可以用来表示“一张图像应该被视为一张没有多级渐远层级（mipmapping levels）的二维纹理”。

在本章中，我们会实现一个 `create_swapchain_image_views` 函数，来为交换链中的每张图像创建一个基本的图像视图，这样我们就可以在之后的章节中将它们用作渲染目标。

首先，在 `AppData` 结构体中添加一个字段，用来存储图像视图：

```rust,noplaypen
struct AppData {
    // ...
    swapchain_image_views: Vec<vk::ImageView>,
}

```

创建一个 `create_swapchain_image_views` 函数，并在 `App::create` 中创建完交换链之后调用它：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_swapchain(window, &instance, &device, &mut data)?;
        create_swapchain_image_views(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_swapchain_image_views(
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

接着，我们实现 `create_swapchain_image_views` 函数，遍历交换链图像，并为每一张图像创建图像视图：

```rust,noplaypen
unsafe fn create_swapchain_image_views(
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    data.swapchain_image_views = data
        .swapchain_images
        .iter()
        .map(|i| {

        })
        .collect::<Result<Vec<_>, _>>()?;

    Ok(())
}
```

对于我们要创建的每一个图像视图，我们首先定义它的颜色分量映射。这允许你对颜色通道进行重新排序。例如，你可以将所有通道映射到红色通道，从而创建一个单色纹理。你也可以将常量值 `0` 和 `1` 映射到通道上。在我们的例子中，我们将使用默认的映射：

```rust,noplaypen
let components = vk::ComponentMapping::builder()
    .r(vk::ComponentSwizzle::IDENTITY)
    .g(vk::ComponentSwizzle::IDENTITY)
    .b(vk::ComponentSwizzle::IDENTITY)
    .a(vk::ComponentSwizzle::IDENTITY);
```

<!-- TODO subresource 这个翻译还有待考虑 -->
接着，我们为图像视图定义子资源（subresource）范围，它描述了图像的用途以及应该访问图像的哪一部分。这里，我们的图像将被用作没有多级渐远层级，也没有多个层次的颜色目标：

```rust,noplaypen
let subresource_range = vk::ImageSubresourceRange::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .base_mip_level(0)
    .level_count(1)
    .base_array_layer(0)
    .layer_count(1);
```

如果你在编写一个立体 3D 应用，那么你可以创建一个包含多个层次的交换链图像视图。然后你可以访问不同的层次，并分别为左眼和右眼的视角创建各自的图像视图。

现在，我们创建一个 `vk::ImageViewCreateInfo` 结构体来提供创建图像视图所需的参数：

```rust,noplaypen
let info = vk::ImageViewCreateInfo::builder()
    .image(*i)
    .view_type(vk::ImageViewType::_2D)
    .format(data.swapchain_format)
    .components(components)
    .subresource_range(subresource_range);
```

`view_type` 和 `format` 字段指定图像数据应该如何被解释。`view_type` 字段用于指定图像应该被视为一维纹理、二维纹理、三维纹理还是立方体贴图。

接下来就只要调用 `create_image_view` 函数了：

```rust,noplaypen
device.create_image_view(&info, None)
```

不同于交换链中的图像，图像视图是由我们显式创建的，所以我们需要在 `App::destroy` 中添加一个类似的循环来销毁它们：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.swapchain_image_views
        .iter()
        .for_each(|v| self.device.destroy_image_view(*v, None));
    // ...
}
```

图像视图已经足以让我们把图像作为纹理使用了，但它还不能用作渲染目标。这还需要一个额外的间接步骤 —— *帧缓冲*（framebuffer）。但在这之前我们需要先建立图形管线。
