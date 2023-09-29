# 图像视图与采样器

> 原文链接：<https://kylemayes.github.io/vulkanalia/texture/image_view_and_sampler.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/24_sampler.rs)

In this chapter we're going to create two more resources that are needed for the graphics pipeline to sample an image. The first resource is one that we've already seen before while working with the swapchain images, but the second one is new - it relates to how the shader will read texels from the image.

在本章中，我们将会创建两个新的资源，这些资源是图形管线中采样图像所必需的。第一个资源我们在之前使用交换链图像时已经见过了，但是第二个资源是全新的，它与着色器如何从图像中读取像素有关。

## 纹理图像视图

We've seen before, with the swapchain images and the framebuffer, that images are accessed through image views rather than directly. We will also need to create such an image view for the texture image.

正如我们之前在交换链图像和帧缓冲中所见，图像不能直接访问，而是要通过图像视图来访问。我们也需要为纹理图像创建这样的图像视图。

Add an `AppData` field to hold a `vk::ImageView` for the texture image and create a new function `create_texture_image_view` where we'll create it:

在 `AppData` 中添加一个 `vk::ImageView` 字段来保存纹理图像的图像视图，并创建一个新的函数 `create_texture_image_view` 来创建它：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_texture_image(&instance, &device, &mut data)?;
        create_texture_image_view(&device, &mut data)?;
        // ...
    }
}

struct AppData {
    // ...
    texture_image: vk::Image,
    texture_image_memory: vk::DeviceMemory,
    texture_image_view: vk::ImageView,
    // ...
}

unsafe fn create_texture_image_view(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

The code for this function can be based directly on `create_swapchain_image_views`. The only two changes you have to make are the `format` and the `image`:

这个函数的代码可以直接基于 `create_swapchain_image_views`。你只需要修改 `format` 和 `image`：

```rust,noplaypen
let subresource_range = vk::ImageSubresourceRange::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .base_mip_level(0)
    .level_count(1)
    .base_array_layer(0)
    .layer_count(1);

let info = vk::ImageViewCreateInfo::builder()
    .image(data.texture_image)
    .view_type(vk::ImageViewType::_2D)
    .format(vk::Format::R8G8B8A8_SRGB)
    .subresource_range(subresource_range);
```

I've left out the explicit `components` initialization, because `vk::ComponentSwizzle::IDENTITY` is defined as `0` anyway. Finish creating the image view by calling `create_image_view`:

我省略了显式的 `components` 初始化，因为 `vk::ComponentSwizzle::IDENTITY` 已经定义为 `0` 了。通过调用 `create_image_view` 来完成图像视图的创建：

```rust,noplaypen
data.texture_image_view = device.create_image_view(&info, None)?;
```

Because so much of the logic is duplicated from `create_swapchain_image_views`, you may wish to abstract it into a new `create_image_view` function:

这里的很多逻辑都和 `create_swapchain_image_views` 重复了，你可能希望将这些逻辑抽出来，写成一个新的 `create_image_view` 函数：

```rust,noplaypen
unsafe fn create_image_view(
    device: &Device,
    image: vk::Image,
    format: vk::Format,
) -> Result<vk::ImageView> {
    let subresource_range = vk::ImageSubresourceRange::builder()
        .aspect_mask(vk::ImageAspectFlags::COLOR)
        .base_mip_level(0)
        .level_count(1)
        .base_array_layer(0)
        .layer_count(1);

    let info = vk::ImageViewCreateInfo::builder()
        .image(image)
        .view_type(vk::ImageViewType::_2D)
        .format(format)
        .subresource_range(subresource_range);

    Ok(device.create_image_view(&info, None)?)
}
```

The `create_texture_image_view` function can now be simplified to:

有了这个函数，`create_texture_image_view` 函数就可以简化为：

```rust,noplaypen
unsafe fn create_texture_image_view(device: &Device, data: &mut AppData) -> Result<()> {
    data.texture_image_view = create_image_view(
        device,
        data.texture_image,
        vk::Format::R8G8B8A8_SRGB,
    )?;

    Ok(())
}
```

And `create_swapchain_image_views` can be simplified to:

而 `create_swapchain_image_views` 可以简化为：

```rust,noplaypen
unsafe fn create_swapchain_image_views(device: &Device, data: &mut AppData) -> Result<()> {
    data.swapchain_image_views = data
        .swapchain_images
        .iter()
        .map(|i| create_image_view(device, *i, data.swapchain_format))
        .collect::<Result<Vec<_>, _>>()?;

    Ok(())
}
```

Make sure to destroy the image view at the end of the program, right before destroying the image itself:

记得在程序结束时，在销毁图像之前销毁图像视图：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_image_view(self.data.texture_image_view, None);
    // ...
}
```

## 采样器

It is possible for shaders to read texels directly from images, but that is not very common when they are used as textures. Textures are usually accessed through samplers, which will apply filtering and transformations to compute the final color that is retrieved.

着色器可以直接从图像中读取纹素，但当图像被用作纹理时，这种做法并不常见。纹理通常是通过采样器访问的，采样器会应用过滤和变换来计算最终的颜色。

These filters are helpful to deal with problems like oversampling. Consider a texture that is mapped to geometry with more fragments than texels. If you simply took the closest texel for the texture coordinate in each fragment, then you would get a result like the first image:

这些过滤器有助于解决采样过密（oversampling）的问题。考虑一个纹理，它被映射到的几何体的片段数比纹素数多。如果你只是简单地为每个片段的纹理坐标取最近的纹素，那么你会得到如下第一张图像的结果：

![](../images/texture_filtering.png)

If you combined the 4 closest texels through linear interpolation, then you would get a smoother result like the one on the right. Of course your application may have art style requirements that fit the left style more (think Minecraft), but the right is preferred in conventional graphics applications. A sampler object automatically applies this filtering for you when reading a color from the texture.

如果你用线性插值结合了最近的 4 个纹素，那么你会得到一个更平滑的结果，如右图所示。当然，你的应用程序可能有更适合左图风格的艺术风格要求（想想 Minecraft），但是在传统的图形应用程序中，右图更受欢迎。当从纹理中读取颜色时，采样器对象会自动为你应用这种过滤。

Undersampling is the opposite problem, where you have more texels than fragments. This will lead to artifacts when sampling high frequency patterns like a checkerboard texture at a sharp angle:

而当纹素数量多于片段时，就会出现采样过疏（undersampling）的问题。当以锐角采样棋盘纹理这样的高频图案时，就会出现伪影：

![](../images/anisotropic_filtering.png)

As shown in the left image, the texture turns into a blurry mess in the distance. The solution to this is [anisotropic filtering](https://en.wikipedia.org/wiki/Anisotropic_filtering), which can also be applied automatically by a sampler.

在左图中，纹理在远处变成了一团模糊的东西。解决这个问题的方法是[各向异性过滤](https://en.wikipedia.org/wiki/Anisotropic_filtering)，采样器也可以自动应用它。

Aside from these filters, a sampler can also take care of transformations. It determines what happens when you try to read texels outside the image through its *addressing mode*. The image below displays some of the possibilities:

除了这些过滤器之外，采样器还可以处理变换。它决定了当你尝试通过其*寻址模式（addressing mode）*读取图像外的纹素时会发生什么。下面的图像展示了一些可能性：

![](../images/texture_addressing.png)

We will now create a function `create_texture_sampler` to set up such a sampler object. We'll be using that sampler to read colors from the texture in the shader later on.

现在我们创建一个 `create_texture_sampler` 函数来设置一个采样器对象。之后我们会在着色器中使用这个采样器来从纹理中读取颜色。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_texture_image(&instance, &device, &mut data)?;
        create_texture_image_view(&device, &mut data)?;
        create_texture_sampler(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_texture_sampler(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

Samplers are configured through a `vk::SamplerCreateInfo` structure, which specifies all filters and transformations that it should apply.

采样器是通过 `vk::SamplerCreateInfo` 结构进行配置的，它指定了它应该应用的所有过滤器和变换。

```rust,noplaypen
let info = vk::SamplerCreateInfo::builder()
    .mag_filter(vk::Filter::LINEAR)
    .min_filter(vk::Filter::LINEAR)
    // continued...
```

The `mag_filter` and `min_filter` fields specify how to interpolate texels that are magnified or minified. Magnification concerns the oversampling problem describes above, and minification concerns undersampling. The choices are `vk::Filter::NEAREST` and `vk::Filter::LINEAR`, corresponding to the modes demonstrated in the images above.

`mag_filter` 和 `min_filter` 字段指定了如何插值放大或缩小的纹素。放大涉及上面描述的采样过密问题，而缩小涉及到采样过疏。选择项是 `vk::Filter::NEAREST` 和 `vk::Filter::LINEAR`，对应于上面图像中演示的模式。

```rust,noplaypen
    .mipmap_mode(vk::SamplerMipmapMode::LINEAR)
```

```rust,noplaypen
    .address_mode_u(vk::SamplerAddressMode::REPEAT)
    .address_mode_v(vk::SamplerAddressMode::REPEAT)
    .address_mode_w(vk::SamplerAddressMode::REPEAT)
```

The addressing mode can be specified per axis using the `address_mode` fields. The available values are listed below. Most of these are demonstrated in the image above. Note that the axes are called U, V and W instead of X, Y and Z. This is a convention for texture space coordinates.

寻址模式可以使用 `address_mode` 字段按轴指定。可用的值如下所示。上面的图像演示了其中的大多数。请注意，这些轴被称为 U、V 和 W，而不是 X、Y 和 Z。这是纹理空间坐标的约定。

* `vk::SamplerAddressMode::REPEAT` &ndash; Repeat the texture when going beyond the image dimensions.
* `vk::SamplerAddressMode::MIRRORED_REPEAT` &ndash; Like repeat, but inverts the coordinates to mirror the image when going beyond the dimensions.
* `vk::SamplerAddressMode::CLAMP_TO_EDGE` &ndash; Take the color of the edge closest to the coordinate beyond the image dimensions.
* `vk::SamplerAddressMode::MIRROR_CLAMP_TO_EDGE` &ndash; Like clamp to edge, but instead uses the edge opposite to the closest edge.
* `vk::SamplerAddressMode::CLAMP_TO_BORDER` &ndash; Return a solid color when sampling beyond the dimensions of the image.

* `vk::SamplerAddressMode::REPEAT` &ndash; 超出图像尺寸时重复纹理。
* `vk::SamplerAddressMode::MIRRORED_REPEAT` &ndash; 类似于重复，但是在超出尺寸时反转坐标以镜像图像。
* `vk::SamplerAddressMode::CLAMP_TO_EDGE` &ndash; 超出图像尺寸时取最接近坐标的边缘的颜色。
* `vk::SamplerAddressMode::MIRROR_CLAMP_TO_EDGE` &ndash; 类似于 clamp to edge，但是使用最接近边缘的相反边缘。
* `vk::SamplerAddressMode::CLAMP_TO_BORDER` &ndash; 超出图像尺寸时返回一个纯色。

It doesn't really matter which addressing mode we use here, because we're not going to sample outside of the image in this tutorial. However, the repeat mode is probably the most common mode, because it can be used to tile textures like floors and walls.

这里我们用什么寻址模式都无所谓，因为在本教程中我们不会采样到图像外面。不过，重复模式可能是最常见的模式，因为它可以用来平铺纹理，比如地板和墙壁。

```rust,noplaypen
    .anisotropy_enable(true)
    .max_anisotropy(16.0)
```

These two fields specify if anisotropic filtering should be used. There is no reason not to use this unless performance is a concern. The `max_anisotropy` field limits the amount of texel samples that can be used to calculate the final color. A lower value results in better performance, but lower quality results. There is no graphics hardware available today that will use more than 16 samples, because the difference is negligible beyond that point.

如果要使用各向异性过滤，我们需要指定两个字段。除非遇到了性能问题，不然没理由不启用各向异性过滤。`max_anisotropy` 字段限制了用于计算最终颜色的纹素样本的数量。将其设为较低的值能带来更好的性能，但也会降低渲染的质量。目前没有任何图形硬件会使用超过 16 个样本，因为就算使用了超过这个数量的样本，也不会有什么区别。

```rust,noplaypen
    .border_color(vk::BorderColor::INT_OPAQUE_BLACK)
```

The `border_color` field specifies which color is returned when sampling beyond the image with clamp to border addressing mode. It is possible to return black, white or transparent in either float or int formats. You cannot specify an arbitrary color.

`border_color` 字段指定了在使用 clamp to border 寻址模式时，采样超出图像范围时返回的颜色。可以返回黑色、白色或透明色，可以是浮点或整数格式。不能指定任意颜色。

```rust,noplaypen
    .unnormalized_coordinates(false)
```

The `unnormalized_coordinates` field specifies which coordinate system you want to use to address texels in an image. If this field is `true`, then you can simply use coordinates within the `[0, width)` and `[0, height)` range. If it is `false`, then the texels are addressed using the `[0, 1)` range on all axes. Real-world applications almost always use normalized coordinates, because then it's possible to use textures of varying resolutions with the exact same coordinates.

`unnormalized_coordinates` 字段指定了你想要使用哪种坐标系来寻址图像中的纹素。如果这个字段是 `true`，那么你可以简单地使用 `[0, width)` 和 `[0, height)` 范围内的坐标。如果是 `false`，那么纹素将使用所有轴上的 `[0, 1)` 范围来寻址。现实世界中的应用几乎总是使用标准化坐标，因为这样就可以用完全相同的坐标使用不同分辨率的纹理。

```rust,noplaypen
    .compare_enable(false)
    .compare_op(vk::CompareOp::ALWAYS)
```

If a comparison function is enabled, then texels will first be compared to a value, and the result of that comparison is used in filtering operations. This is mainly used for [percentage-closer filtering](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html) on shadow maps. We'll look at this in a future chapter.

如果启用了比较函数，那么纹素将首先与一个值进行比较，比较的结果将用于过滤操作。这主要用于阴影贴图上的[近似百分比过滤](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)。我们将在后面的章节中介绍这个。

```rust,noplaypen
    .mipmap_mode(vk::SamplerMipmapMode::LINEAR)
    .mip_lod_bias(0.0)
    .min_lod(0.0)
    .max_lod(0.0);
```

All of these fields apply to mipmapping. We will look at mipmapping in a [later chapter](/Generating_Mipmaps), but basically it's another type of filter that can be applied.

接下来的这些字段都与多级渐远有关。我们将在[后面的章节](/Generating_Mipmaps)中介绍多级渐远，但简单来说它就是另一种过滤器。

The functioning of the sampler is now fully defined. Add an `AppData` field to hold the handle of the sampler object:

采样器的功能现在已经完全定义了。添加一个 `AppData` 字段来保存采样器对象的句柄：

```rust,noplaypen
struct AppData {
    // ...
    texture_image_view: vk::ImageView,
    texture_sampler: vk::Sampler,
    // ...
}
```

Then create the sampler with `create_sampler`:

接着使用 `create_sampler` 函数来创建这个采样器：

```rust,noplaypen
data.texture_sampler = device.create_sampler(&info, None)?;
```

Note the sampler does not reference a `vk::Image` anywhere. The sampler is a distinct object that provides an interface to extract colors from a texture. It can be applied to any image you want, whether it is 1D, 2D or 3D. This is different from many older APIs, which combined texture images and filtering into a single state.

注意采样器不引用 `vk::Image`。采样器是一个独立的对象，它提供了从纹理中提取颜色的接口。它可以应用于任何你想要的图像，无论是 1D、2D 还是 3D。这与许多旧的 API 不同，旧的 API 将纹理图像和过滤器组合成一个单一的状态。

Destroy the sampler at the end of the program when we'll no longer be accessing the image:

在程序结束、我们不再访问图像之后销毁采样器：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_sampler(self.data.texture_sampler, None);
    // ...
}
```

## 各向异性设备特性

If you run your program right now, you'll see a validation layer message like this:

如果你现在运行程序，你会看到一条类似这样的校验层消息：

![](../images/validation_layer_anisotropy.png)

That's because anisotropic filtering is actually an optional device feature. We need to update the `create_logical_device` function to request it:

这是因为各向异性过滤其实是一项可选的设备特性。我们需要更新 `create_logical_device` 函数来请求它：

```rust,noplaypen
let features = vk::PhysicalDeviceFeatures::builder()
    .sampler_anisotropy(true);
```

And even though it is very unlikely that a modern graphics card will not support it, we should update `check_physical_device` to check if it is available:

并且尽管现代显卡不支持各向异性过滤的可能性非常小，但我们还是应该更新 `check_physical_device` 来检查它是否可用：

```rust,noplaypen
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    // ...

    let features = instance.get_physical_device_features(physical_device);
    if features.sampler_anisotropy != vk::TRUE {
        return Err(anyhow!(SuitabilityError("No sampler anisotropy.")));
    }

    Ok(())
}
```

The `get_physical_device_features` repurposes the `vk::PhysicalDeviceFeatures` struct to indicate which features are supported rather than requested by setting the boolean values.

`get_physical_device_features` 重用了 `vk::PhysicalDeviceFeatures` 结构体，通过设置布尔值来指示支持哪些特性，而不是请求哪些特性。

Instead of enforcing the availability of anisotropic filtering, it's also possible to simply not use it by conditionally setting:

你也可以根据条件来设置，而不是强制使用各向异性过滤：

```rust,noplaypen
    .anisotropy_enable(false)
    .max_anisotropy(1.0)
```

In the next chapter we will expose the image and sampler objects to the shaders to draw the texture onto the square.

下一章中，我们将把图像和采样器对象暴露给着色器，以便将纹理绘制到正方形上。
