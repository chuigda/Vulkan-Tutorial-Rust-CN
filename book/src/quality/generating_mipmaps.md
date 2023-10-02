# 生成多级渐远

> 原文链接：<https://kylemayes.github.io/vulkanalia/quality/generating_mipmaps.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/28_mipmapping.rs)

Our program can now load and render 3D models. In this chapter, we will add one more feature, mipmap generation. Mipmaps are widely used in games and rendering software, and Vulkan gives us complete control over how they are created.

现在我们的程序可以加载并渲染 3D 模型了。本章中我们会再添加一个特性：多级渐远生成。多级渐远被广泛用于游戏和渲染软件中，而 Vulkan 允许你完全地控制多级渐远的创建方式。

Mipmaps are precalculated, downscaled versions of an image. Each new image is half the width and height of the previous one.  Mipmaps are used as a form of *Level of Detail* or *LOD.* Objects that are far away from the camera will sample their textures from the smaller mip images. Using smaller images increases the rendering speed and avoids artifacts such as [Moiré patterns](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern). An example of what mipmaps look like:

多级渐远是预先计算好的、缩小过的图像。每个新图像的宽度和高度都是上一个图像的一半。多级渐远被用作*细节级别（Level of Detail, LOD）*一种形式。远离相机的物体将从较小的多级渐远图像中采样它们的纹理。使用较小的图像可以提高渲染速度并避免诸如[摩尔纹](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern)之类的伪像。下面是多级渐远的一个例子：

![](../images/mipmaps_example.jpg)

## 创建图像

In Vulkan, each of the mip images is stored in different *mip levels* of a `vk::Image`. Mip level 0 is the original image, and the mip levels after level 0 are commonly referred to as the *mip chain.*

在 Vulkan 中，每个多级渐远图像都存储在 `vk::Image` 的不同*多级渐远级别*中。多级渐远级别 0 是原始图像，级别 0 之后的多级渐远级别通常被称为*多级渐远链*。

The number of mip levels is specified when the `vk::Image` is created. Up until now, we have always set this value to one. We need to calculate the number of mip levels from the dimensions of the image. First, add an `AppData` field to store this number:

多级渐远级别的数量在创建 `vk::Image` 时指定。到目前为止，我们总是将这个值设置为 1，而现在我们需要根据图像的尺寸计算多级渐远级别的数量。首先，在 `AppData` 中添加一个字段来存储这个数字：

```rust,noplaypen
struct AppData {
    // ...
    mip_levels: u32,
    texture_image: vk::Image,
    // ...
}
```

The value for `mip_levels` can be found once we've loaded the texture in `create_texture_image`:

`mip_levels` 的值可以在 `create_texture_image` 中加载纹理后被计算出来：

```rust,noplaypen
let image = File::open("resources/viking_room.png")?;

let decoder = png::Decoder::new(image);
let mut reader = decoder.read_info()?;

// ...

data.mip_levels = (width.max(height) as f32).log2().floor() as u32 + 1;
```

This calculates the number of levels in the mip chain. The `max` method selects the largest dimension. The `log2` method calculates how many times that dimension can be divided by 2. The `floor` method handles cases where the largest dimension is not a power of 2. `1` is added so that the original image has a mip level.

这个表达式计算了多级渐远链中的级别数量。`max` 方法选择长和宽中更大的维度。`log2` 方法计算该维度可以被 2 整除的次数。`floor` 方法处理该维度不是 2 的幂的情况。最后为了让原始图像也有一个多级渐远级别，我们加上 `1`。

To use this value, we need to change the `^create_image`, `^create_image_view`, and `transition_image_layout` functions to allow us to specify the number of mip levels. Add a `mip_levels` parameter to the functions:

要使用这个值，我们需要修改 `create_image`、`create_image_view` 和 `transition_image_layout` 函数，以允许我们指定多级渐远级别的数量。在这些函数中添加一个 `mip_levels` 参数：

```rust,noplaypen
unsafe fn create_image(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    width: u32,
    height: u32,
    mip_levels: u32,
    format: vk::Format,
    tiling: vk::ImageTiling,
    usage: vk::ImageUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> Result<(vk::Image, vk::DeviceMemory)> {
    let info = vk::ImageCreateInfo::builder()
        // ...
        .mip_levels(mip_levels)
        // ...

    // ...
}
```

```rust,noplaypen
unsafe fn create_image_view(
    device: &Device,
    image: vk::Image,
    format: vk::Format,
    aspects: vk::ImageAspectFlags,
    mip_levels: u32,
) -> Result<vk::ImageView> {
    let subresource_range = vk::ImageSubresourceRange::builder()
        // ...
        .level_count(mip_levels)
        // ...

    // ...
}
```

```rust,noplaypen
unsafe fn transition_image_layout(
    device: &Device,
    data: &AppData,
    image: vk::Image,
    format: vk::Format,
    old_layout: vk::ImageLayout,
    new_layout: vk::ImageLayout,
    mip_levels: u32,
) -> Result<()> {
    // ...

    let subresource = vk::ImageSubresourceRange::builder()
        // ...
        .level_count(mip_levels)
        // ...

    // ...
}
```

Update all calls to these functions to use the right values:

更新所有调用这些函数的地方，以使用正确的值：

> Note: Be sure to use a value of `1` for all of the images and image views except the image and image view that is for the texture.

> 注意：确保对除纹理之外的所有图像和图像视图使用 `1`。

```rust,noplaypen
let (depth_image, depth_image_memory) = create_image(
    instance,
    device,
    data,
    data.swapchain_extent.width,
    data.swapchain_extent.height,
    1,
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
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::SAMPLED | vk::ImageUsageFlags::TRANSFER_DST,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
)?;
```

```rust,noplaypen
create_image_view(
    device,
    *i,
    data.swapchain_format,
    vk::ImageAspectFlags::COLOR,
    1,
)

// ...

data.depth_image_view = create_image_view(
    device,
    data.depth_image,
    format,
    vk::ImageAspectFlags::DEPTH,
    1,
)?;

// ...

data.texture_image_view = create_image_view(
    device,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageAspectFlags::COLOR,
    data.mip_levels,
)?;
```

```rust,noplaypen
transition_image_layout(
    device,
    data,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::UNDEFINED,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    data.mip_levels,
)?;

// ...

transition_image_layout(
    device,
    data,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
    data.mip_levels,
)?;
```

## 生成多级渐远

Our texture image now has multiple mip levels, but the staging buffer can only be used to fill mip level 0. The other levels are still undefined. To fill these levels we need to generate the data from the single level that we have. We will use the `cmd_blit_image` command. This command performs copying, scaling, and filtering operations. We will call this multiple times to *blit* data to each level of our texture image.

现在我们的纹理图像有多个多级渐远级别了，但是暂存缓冲区只能用来填充级别 0。其他级别仍然是未定义的。要填充这些级别，我们需要从我们拥有的单个级别生成数据。我们将使用 `cmd_blit_image` 命令。这个命令执行复制、缩放和过滤操作。我们将多次调用它来将数据*blit*到我们纹理图像的每个级别。

`cmd_blit_image` is considered a transfer operation, so we must inform Vulkan that we intend to use the texture image as both the source and destination of a transfer. Add `vk::ImageUsageFlags::TRANSFER_SRC` to the texture image's usage flags in `create_texture_image`:

`cmd_blit_image` 被视为一个传输操作，所以我们必须告诉 Vulkan 我们打算将纹理图像用作传输源和传输目标。在 `create_texture_image` 中，将 `vk::ImageUsageFlags::TRANSFER_SRC` 添加到纹理图像的用法标志中：

```rust,noplaypen
let (texture_image, texture_image_memory) = create_image(
    instance,
    device,
    data,
    width,
    height,
    data.mip_levels,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::SAMPLED
        | vk::ImageUsageFlags::TRANSFER_DST
        | vk::ImageUsageFlags::TRANSFER_SRC,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
)?;
```

Like other image operations, `cmd_blit_image` depends on the layout of the image it operates on. We could transition the entire image to `vk::ImageLayout::GENERAL`, but this will most likely be slow. For optimal performance, the source image should be in `vk::ImageLayout::TRANSFER_SRC_OPTIMAL` and the destination image should be in `vk::ImageLayout::TRANSFER_DST_OPTIMAL`. Vulkan allows us to transition each mip level of an image independently. Each blit will only deal with two mip levels at a time, so we can transition each level into the optimal layout between blits commands.

和其他图像操作一样，`cmd_blit_image` 依赖于图像的布局。我们可以将整个图像转换为 `vk::ImageLayout::GENERAL`，但这很可能会很慢。为了获得最佳性能，源图像应该是 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`，目标图像应该是 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。Vulkan 允许我们独立地转换图像的每个多级渐远级别。每个 blit 每次只处理两个多级渐远级别，所以我们可以在 blit 命令之间将每个级别转换为最佳布局。

`transition_image_layout` only performs layout transitions on the entire image, so we'll need to write a few more pipeline barrier commands. Remove the existing transition to `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL` in `create_texture_image`.

`transition_image_layout` 只对整个图像执行布局转换，所以我们需要再写几个管线屏障命令。在 `create_texture_image` 中删除现有的将图像转换到 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL` 的操作。

This will leave each level of the texture image in `vk::ImageLayout::TRANSFER_DST_OPTIMAL`. Each level will be transitioned to `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL` after the blit command reading from it is finished.

这将使纹理图像的每个级别都处于 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。在从中读取的 blit 命令完成后，每个级别将转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。

We're now going to write the function that generates the mipmaps:

现在我们要写生成多级渐远的函数：

```rust,noplaypen
unsafe fn generate_mipmaps(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    image: vk::Image,
    width: u32,
    height: u32,
    mip_levels: u32,
) -> Result<()> {
    let command_buffer = begin_single_time_commands(device, data)?;

    let subresource = vk::ImageSubresourceRange::builder()
        .aspect_mask(vk::ImageAspectFlags::COLOR)
        .base_array_layer(0)
        .layer_count(1)
        .level_count(1);

    let mut barrier = vk::ImageMemoryBarrier::builder()
        .image(image)
        .src_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .dst_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .subresource_range(subresource);

    end_single_time_commands(device, data, command_buffer)?;

    Ok(())
}
```

We're going to make several transitions, so we'll reuse this `vk::ImageMemoryBarrier` (which is why it is defined as mutable). The fields set above will remain the same for all barriers. `subresource_range.mip_level`, `old_layout`, `new_layout`, `src_access_mask`, and `dst_access_mask` will be changed for each transition.

我们将进行多次转换，所以我们将重用这个 `vk::ImageMemoryBarrier`（这就是为什么它被定义为可变的）。上面设置的字段将对所有屏障保持不变。`subresource_range.mip_level`、`old_layout`、`new_layout`、`src_access_mask` 和 `dst_access_mask` 将在每次转换中被改变。

```rust,noplaypen
let mut mip_width = width;
let mut mip_height = height;

for i in 1..mip_levels {
}
```

This loop will record each of the `cmd_blit_image` commands. Note that the range index starts at 1, not 0.

这个循环将记录每个 `cmd_blit_image` 命令。注意，范围索引从 1 开始，而不是 0。

```rust,noplaypen
barrier.subresource_range.base_mip_level = i - 1;
barrier.old_layout = vk::ImageLayout::TRANSFER_DST_OPTIMAL;
barrier.new_layout = vk::ImageLayout::TRANSFER_SRC_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_WRITE;
barrier.dst_access_mask = vk::AccessFlags::TRANSFER_READ;

device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::TRANSFER,
    vk::DependencyFlags::empty(),
    &[] as &[vk::MemoryBarrier],
    &[] as &[vk::BufferMemoryBarrier],
    &[barrier],
);
```

First, we transition level `i - 1` to `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`. This transition will wait for level `i - 1` to be filled, either from the previous blit command, or from `cmd_copy_buffer_to_image`. The current blit command will wait on this transition.

首先，我们将级别 `i - 1` 转换为 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`。这个转换将等待级别 `i - 1` 被填充，要么是来自前一个 blit 命令，要么是来自 `cmd_copy_buffer_to_image`。当前的 blit 命令将等待这个转换。

```rust,noplaypen
let src_subresource = vk::ImageSubresourceLayers::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .mip_level(i - 1)
    .base_array_layer(0)
    .layer_count(1);

let dst_subresource = vk::ImageSubresourceLayers::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .mip_level(i)
    .base_array_layer(0)
    .layer_count(1);

let blit = vk::ImageBlit::builder()
    .src_offsets([
        vk::Offset3D { x: 0, y: 0, z: 0 },
        vk::Offset3D {
            x: mip_width as i32,
            y: mip_height as i32,
            z: 1,
        },
    ])
    .src_subresource(src_subresource)
    .dst_offsets([
        vk::Offset3D { x: 0, y: 0, z: 0 },
        vk::Offset3D {
            x: (if mip_width > 1 { mip_width / 2 } else { 1 }) as i32,
            y: (if mip_height > 1 { mip_height / 2 } else { 1 }) as i32,
            z: 1,
        },
    ])
    .dst_subresource(dst_subresource);
```

Next, we specify the regions that will be used in the blit operation. The source mip level is `i - 1` and the destination mip level is `i`. The two elements of the `src_offsets` array determine the 3D region that data will be blitted from. `dst_offsets` determines the region that data will be blitted to. The X and Y dimensions of the `dst_offsets[1]` are divided by two since each mip level is half the size of the previous level. The Z dimension of `src_offsets[1]` and `dst_offsets[1]` must be 1, since a 2D image has a depth of 1.

接着，我们指定 blit 操作将会使用的区域。源多级渐远级别是 `i - 1`，目标多级渐远级别是 `i`。`src_offsets` 数组的两个元素决定了数据将从哪里 blit 出来的 3D 区域。`dst_offsets` 决定了数据将 blit 到哪里的区域。`dst_offsets[1]` 的 X 和 Y 维度被 2 除以，因为每个多级渐远级别的大小是前一个级别的一半。`src_offsets[1]` 和 `dst_offsets[1]` 的 Z 维度必须是 1，因为 2D 图像的深度是 1。

```rust,noplaypen
device.cmd_blit_image(
    command_buffer,
    image,
    vk::ImageLayout::TRANSFER_SRC_OPTIMAL,
    image,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    &[blit],
    vk::Filter::LINEAR,
);
```

Now, we record the blit command. Note that `image` is used for both the `stc_image` and `dst_image` parameters. This is because we're blitting between different levels of the same image. The source mip level was just transitioned to `vk::ImageLayout::TRANSFER_SRC_OPTIMAL` and the destination level is still in `vk::ImageLayout::TRANSFER_DST_OPTIMAL` from `create_texture_image`.

现在，我们记录 blit 命令。注意，`image` 被用于 `src_image` 和 `dst_image` 参数。这是因为我们在同一图像的不同级别之间 blit。源多级渐远级别刚刚转换为 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`，而目标级别仍然处于 `create_texture_image` 中的 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。

Beware if you are using a dedicated transfer queue (as suggested in the `Vertex buffers` chapter): `cmd_blit_image` must be submitted to a queue with graphics capability.

如果你正在使用专用的传输队列（如 `顶点缓冲` 章节中所建议的）：`cmd_blit_image` 必须提交到具有图形功能的队列。

The last parameter allows us to specify a `vk::Filter` to use in the blit. We have the same filtering options here that we had when making the `vk::Sampler`. We use the `vk::Filter::LINEAR` to enable interpolation.

最后一个参数允许我们指定 blit 中使用的 `vk::Filter`。我们在这里有与创建 `vk::Sampler` 时相同的过滤选项。我们使用 `vk::Filter::LINEAR` 来启用插值。

```rust,noplaypen
barrier.old_layout = vk::ImageLayout::TRANSFER_SRC_OPTIMAL;
barrier.new_layout = vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_READ;
barrier.dst_access_mask = vk::AccessFlags::SHADER_READ;

device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::FRAGMENT_SHADER,
    vk::DependencyFlags::empty(),
    &[] as &[vk::MemoryBarrier],
    &[] as &[vk::BufferMemoryBarrier],
    &[barrier],
);
```

This barrier transitions mip level `i - 1` to `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`. This transition waits on the current blit command to finish. All sampling operations will wait on this transition to finish.

这个屏障将多级渐远级别 `i - 1` 转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。这个转换等待当前的 blit 命令完成。所有采样操作都将等待这个转换完成。

```rust,noplaypen
if mip_width > 1 {
    mip_width /= 2;
}

if mip_height > 1 {
    mip_height /= 2;
}
```

At the end of the loop, we divide the current mip dimensions by two. We check each dimension before the division to ensure that dimension never becomes 0. This handles cases where the image is not square, since one of the mip dimensions would reach 1 before the other dimension. When this happens, that dimension should remain 1 for all remaining levels.

在循环体结束时，我们将当前的多级渐远维度除以 2。我们在除法之前检查每个维度，以确保该维度永远不会变为 0。这处理了图像不是正方形的情况，因为如果图像不是正方形，其中一个多级渐远维度会在另一个维度之前达到 1。当这种情况发生时，在剩下的级别中该维度应该保持 1。

```rust,noplaypen
barrier.subresource_range.base_mip_level = mip_levels - 1;
barrier.old_layout = vk::ImageLayout::TRANSFER_DST_OPTIMAL;
barrier.new_layout = vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_WRITE;
barrier.dst_access_mask = vk::AccessFlags::SHADER_READ;

device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::FRAGMENT_SHADER,
    vk::DependencyFlags::empty(),
    &[] as &[vk::MemoryBarrier],
    &[] as &[vk::BufferMemoryBarrier],
    &[barrier],
);

end_single_time_commands(device, data, command_buffer)?;
```

Before we end the command buffer, we insert one more pipeline barrier. This barrier transitions the last mip level from `vk::ImageLayout::TRANSFER_DST_OPTIMAL` to `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`. This wasn't handled by the loop, since the last mip level is never blitted from.

在我们结束命令缓冲区之前，我们插入了一个管线屏障。这个屏障将最后一个多级渐远级别从 `vk::ImageLayout::TRANSFER_DST_OPTIMAL` 转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。这在循环中没有处理，因为最后一个多级渐远级别不会被 blit 操作。

Finally, add the call to `generate_mipmaps` at the end of `create_texture_image`:

最后，在 `create_texture_image` 的末尾添加对 `generate_mipmaps` 的调用：

```rust,noplaypen
generate_mipmaps(
    instance,
    device,
    data,
    data.texture_image,
    width,
    height,
    data.mip_levels,
)?;
```

Our texture image's mipmaps are now completely filled.

我们的纹理图像的多级渐远现在已经填充好了。

## 线性过滤支持

It is very convenient to use a built-in command like `cmd_blit_image` to generate all the mip levels, but unfortunately it is not guaranteed to be supported on all platforms. It requires the texture image format we use to support linear filtering, which can be checked with the `get_physical_device_format_properties` command. We will add a check to the `generate_mipmaps` function for this.

使用内置命令 `cmd_blit_image` 来生成所有的多级渐远级别非常方便，但不幸的是，并非所有的平台都保证支持它。`cmd_blit_image` 要求我们使用的纹理图像格式支持线性过滤，这可以通过 `get_physical_device_format_properties` 命令来检查。我们将在 `generate_mipmaps` 函数中添加一个检查。

First add an additional parameter that specifies the image format:

首先添加一个额外的参数来指定图像格式：

```rust,noplaypen
generate_mipmaps(
    instance,
    device,
    data,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    width,
    height,
    data.mip_levels,
)?;

// ...

unsafe fn generate_mipmaps(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    image: vk::Image,
    format: vk::Format,
    width: u32,
    height: u32,
    mip_levels: u32,
) -> Result<()> {
    // ...
}
```

In the `generate_mipmaps` function, use `get_physical_device_format_properties` to request the properties of the texture image format and check that linear filtering is supported:

在 `generate_mipmaps` 函数中，使用 `get_physical_device_format_properties` 来请求纹理图像格式的属性，并检查是否支持线性过滤：

```rust,noplaypen
if !instance
    .get_physical_device_format_properties(data.physical_device, format)
    .optimal_tiling_features
    .contains(vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR)
{
    return Err(anyhow!("Texture image format does not support linear blitting!"));
}
```

The `vk::FormatProperties` struct has three fields named `linear_tiling_features`, `optimal_tiling_features`, and `buffer_features` that each describe how the format can be used depending on the way it is used. We create a texture image with the optimal tiling format, so we need to check `optimal_tiling_features`. Support for the linear filtering feature can be checked with `vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR`.

`vk::FormatProperties` 结构有三个字段，分别命名为 `linear_tiling_features`、`optimal_tiling_features` 和 `buffer_features`，它们描述了格式在使用方式不同的情况下的使用方式。我们使用最佳平铺格式创建纹理图像，所以我们需要检查 `optimal_tiling_features`。可以使用 `vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR` 来检查线性过滤特性的支持。

There are two alternatives in the case where this is not supported. You could implement a function that searches common texture image formats for one that *does* support linear blitting, or you could implement the mipmap generation in your software. Each mip level can then be loaded into the image in the same way that you loaded the original image.

在不支持的情况下有两种替代方案。你可以实现一个函数，搜索常见的纹理图像格式，找到一个支持线性 blit 的格式，或者你可以在你的软件中实现多级渐远的生成。然后，每个多级渐远级别可以以与加载原始图像相同的方式加载到图像中。

It should be noted that it is uncommon in practice to generate the mipmap levels at runtime anyway. Usually they are pregenerated and stored in the texture file alongside the base level to improve loading speed. Implementing resizing in software and loading multiple levels from a file is left as an exercise to the reader.

应该指出的是，在实践中，通常不会在运行时生成多级渐远级别。通常，它们是预先生成的，并与基础级别一起存储在纹理文件中，以提高加载速度。在软件中实现调整大小并从文件加载多个级别的功能留给读者作为练习。

## 采样器

While the `vk::Image` holds the mipmap data, `vk::Sampler` controls how that data is read while rendering. Vulkan allows us to specify `min_lod`, `max_lod`, `mip_lod_bias`, and `mipmap_mode` ("LOD" means "Level of Detail"). When a texture is sampled, the sampler selects a mip level according to the following pseudocode:

`vk::Image` 中保存了多级渐远数据，而 `vk::Sampler` 控制了渲染时如何读取这些数据。Vulkan 允许我们指定 `min_lod`、`max_lod`、`mip_lod_bias` 和 `mipmap_mode`（"LOD" 意味着 "细节级别"）。当采样纹理时，采样器根据以下伪代码选择一个多级渐远级别：

```rust,noplaypen
// Smaller when the object is close, may be negative.
let mut lod = get_lod_level_from_screen_size();

lod = clamp(lod + mip_lod_bias, min_lop, max_lod);

// Clamped to the number of mip levels in the texture.
let level = clamp(floor(lod), 0, texture.mip_levels - 1);

let color = if mipmap_mode == vk::SamplerMipmapMode::NEAREST {
    sample(level)
} else {
    blend(sample(level), sample(level + 1))
};
```

If `sampler_info.mipmap_mode` is `vk::SamplerMipmapMode::NEAREST`, `lod` selects the mip level to sample from. If the mipmap mode is `vk::SamplerMipmapMode::LINEAR`, `lod` is used to select two mip levels to be sampled. Those levels are sampled and the results are linearly blended.

如果 `sampler_info.mipmap_mode`（多级渐远模式）是 `vk::SamplerMipmapMode::NEAREST`，则 `lod` 选择要从中采样的多级渐远级别。如果多级渐远模式是 `vk::SamplerMipmapMode::LINEAR`，则 `lod` 用于选择要采样的两个mip级别，对这些级别进行采样，并对结果线性混合。

The sample operation is also affected by `lod`:

采样操作也受 `lod` 的影响：

```rust,noplaypen
let color = if lod <= 0 {
    read_texture(uv, mag_filter)
} else {
    read_texture(uv, min_filter)
};
```

If the object is close to the camera, `mag_filter` is used as the filter. If the object is further from the camera, `min_filter` is used. Normally, `lod` is non-negative, and is only 0 when close the camera. `mip_lod_bias` lets us force Vulkan to use lower `lod` and `level` than it would normally use.

如果物体靠近相机，`mag_filter` 将被用作过滤器。如果物体离相机更远，`min_filter` 将被用作过滤器。通常，`lod` 是非负的，只有在靠近相机时才是 0。`mip_lod_bias` 让我们强制 Vulkan 使用比它通常使用的更低的 `lod` 和 `level`。

To see the results of this chapter, we need to choose values for our `texture_sampler`. We've already set the `min_filter` and `mag_filter` to use `vk::Filter::LINEAR`. We just need to choose values for `min_lod`, `max_lod`, `mip_lod_bias`, and `mipmap_mode`.

为了看到本章代码的效果，我们需要设置 `texture_sampler` 的字段。我们之前已经将 `min_filter` 和 `mag_filter` 设置为 `vk::Filter::LINEAR`，现在我们只需要设置 `min_lod`、`max_lod`、`mip_lod_bias` 和 `mipmap_mode`。

```rust,noplaypen
unsafe fn create_texture_sampler(device: &Device, data: &mut AppData) -> Result<()> {
    let info = vk::SamplerCreateInfo::builder()
        // ...
        .mipmap_mode(vk::SamplerMipmapMode::LINEAR)
        .min_lod(0.0)       // Optional.
        .max_lod(data.mip_levels as f32)
        .mip_lod_bias(0.0); // Optional.

    data.texture_sampler = device.create_sampler(&info, None)?;

    Ok(())
}
```

To allow the full range of mip levels to be used, we set `min_lod` to `0.0` and `max_lod` to the number of mip levels. We have no reason to change the `lod` value, so we set `mip_lod_bias` to 0.0f.

要使用所有的多级渐远级别，我们将 `min_lod` 设置为 `0.0`，`max_lod` 设置为多级渐远级别的数量。我们没有理由改变 `lod` 值，所以我们将 `mip_lod_bias` 设置为 0.0f。

Now run your program and you should see the following:

现在运行你的程序，你应该会看到以下内容：

![](../images/mipmaps.png)

It's not a dramatic difference, since our scene is so simple. There are subtle differences if you look closely (it will be much easier to spot differences if you open the below image in a separate tab so you can see it at full size).

我们的场景太简单了，所以没有什么明显的区别。如果你仔细观察，你会发现微妙的区别（如果你在单独的标签中打开下面的图像，你会发现区别很容易被发现，这样你就可以看到它的全尺寸）。

![](../images/mipmaps_comparison.png)

One of most noticeable differences is the axe head. With mipmaps, the borders between the dark gray and light gray areas have been smoothed. Without mipmaps, these borders are much sharper. The differences are clear in this image which shows the axe head with and without mipmapping at 8x magnification (without any filtering so the pixels are simply expanded).

最明显的区别之一是斧头。有了多级渐远，深灰色和浅灰色区域之间的边界已经被平滑了。要是没有多级渐远，这些边界要锐利得多。在这张图像中，斧头被放大了 8 倍，以显示有多级渐远和没有多级渐远（没有任何过滤，所以像素只是被扩展了）的情况。

![](../images/mipmaps_comparison_axe.png)

You can play around with the sampler settings to see how they affect mipmapping. For example, by changing `min_lod`, you can force the sampler to not use the lowest mip levels:

你可以玩一下采样器的设置，看看它们如何影响多级渐远。例如，通过改变 `min_lod`，你可以强制采样器不使用最低的多级渐远级别：

```rust,noplaypen
.min_lod(data.mip_levels as f32 / 2.0)
```

These settings will produce this image:

这些设置将产生这样的图像：

![](../images/highmipmaps.png)

This is how higher mip levels will be used when objects are further away from the camera.

这是当物体离相机更远时，更高的多级渐远级别将如何使用。
