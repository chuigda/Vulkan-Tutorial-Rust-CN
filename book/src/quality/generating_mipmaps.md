# 生成多级渐远

> 原文链接：<https://kylemayes.github.io/vulkanalia/quality/generating_mipmaps.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/28_mipmapping.rs)

现在我们的程序可以加载并渲染 3D 模型了。本章中我们会再添加一个特性：多级渐远生成。多级渐远被广泛用于游戏和渲染软件中，而 Vulkan 允许你完全地控制多级渐远的创建方式。

多级渐远是预先计算好的、缩小过的图像。每个新图像的宽度和高度都是上一个图像的一半。多级渐远被用作*细节层级（Level of Detail, LOD）*一种形式。远离相机的物体将从较小的多级渐远图像中采样它们的纹理。使用较小的图像可以提高渲染速度并避免诸如[摩尔纹](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern)之类的伪像。下面是多级渐远的一个例子：

![](../images/mipmaps_example.jpg)

## 创建图像

在 Vulkan 中，每个多级渐远图像都存储在 `vk::Image` 的不同*多级渐远层级*中。多级渐远层级 0 是原始图像，层级 0 之后的多级渐远层级通常被称为*多级渐远链*。

多级渐远层级的数量在创建 `vk::Image` 时指定。到目前为止，我们总是将这个值设置为 1，而现在我们需要根据图像的尺寸计算多级渐远层级的数量。首先，在 `AppData` 中添加一个字段来存储这个数字：

```rust,noplaypen
struct AppData {
    // ...
    mip_levels: u32,
    texture_image: vk::Image,
    // ...
}
```

`mip_levels` 的值可以在 `create_texture_image` 中加载纹理后被计算出来：

```rust,noplaypen
let image = File::open("resources/viking_room.png")?;

let decoder = png::Decoder::new(image);
let mut reader = decoder.read_info()?;

// ...

data.mip_levels = (width.max(height) as f32).log2().floor() as u32 + 1;
```

这个表达式计算了多级渐远链中的层级数量。`max` 方法选择长和宽中更大的维度。`log2` 方法计算该维度可以被 2 整除的次数。`floor` 方法处理该维度不是 2 的幂的情况。最后为了让原始图像也有一个多级渐远层级，我们加上 `1`。

要使用这个值，我们需要修改 `create_image`、`create_image_view` 和 `transition_image_layout` 函数，以允许我们指定多级渐远层级的数量。在这些函数中添加一个 `mip_levels` 参数：

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

更新所有调用这些函数的地方，以使用正确的值：

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

现在我们的纹理图像有多个多级渐远层级了，但目前为止我们只用暂存缓冲填充了层级 0。其他层级的内容仍然是未定义的。要填充这些层级，我们需要从我们拥有的单个层级生成数据。我们将使用 `cmd_blit_image` 指令。这个指令执行复制、缩放和过滤操作。我们将多次调用它来将数据*blit*到我们纹理图像的每个层级。

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

和其他图像操作一样，`cmd_blit_image` 依赖于图像的布局。我们可以将整个图像转换为 `vk::ImageLayout::GENERAL`，但这很可能会很慢。为了获得最佳性能，源图像应该是 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`，目标图像应该是 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。Vulkan 允许我们独立地转换图像的每个多级渐远层级。每个 blit 每次只处理两个多级渐远层级，所以我们可以在 blit 指令之间将每个层级转换为最佳布局。

`transition_image_layout` 只对整个图像执行布局转换，所以我们需要再写几个管线屏障指令。在 `create_texture_image` 中删除现有的将图像转换到 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL` 的操作。

这将使纹理图像的每个层级都处于 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。在从中读取的 blit 指令完成后，每个层级将被转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。

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

我们将进行多次转换，所以我们将重用这个 `vk::ImageMemoryBarrier`（这就是为什么它被定义为可变的）。上面设置的字段将对所有屏障保持不变。`subresource_range.mip_level`、`old_layout`、`new_layout`、`src_access_mask` 和 `dst_access_mask` 将在每次转换中被改变。

```rust,noplaypen
let mut mip_width = width;
let mut mip_height = height;

for i in 1..mip_levels {
}
```

This loop will record each of the `cmd_blit_image` commands. Note that the range index starts at 1, not 0.

这个循环将记录每个 `cmd_blit_image` 指令。注意，范围索引从 1 开始，而不是 0。

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

首先，我们将层级 `i - 1` 转换为 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`。这个转换将等待层级`i - 1` 被填充，要么是来自前一个 blit 指令，要么是来自 `cmd_copy_buffer_to_image`。当前的 blit 指令将等待这个转换。

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

接着，我们指定 blit 操作将会使用的区域。源多级渐远层级是 `i - 1`，目标多级渐远层级是 `i`。`src_offsets` 数组的两个元素决定了数据将从哪里 blit 出来的 3D 区域。`dst_offsets` 决定了数据将 blit 到哪里的区域。`dst_offsets[1]` 的 X 和 Y 维度被 2 除以，因为每个多级渐远层级的大小是前一个层级的一半。`src_offsets[1]` 和 `dst_offsets[1]` 的 Z 维度必须是 1，因为 2D 图像的深度是 1。

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

现在，我们记录 blit 指令。注意，`image` 被用于 `src_image` 和 `dst_image` 参数。这是因为我们在同一图像的不同层级之间 blit。源多级渐远层级刚刚转换为 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`，而目标级层级仍然处于 `create_texture_image` 中的 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`。

如果你正在使用专用的传输队列（如 `顶点缓冲` 章节中所提到的），那么请注意 `cmd_blit_image` 必须被提交到具有图形功能的队列。

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

这个屏障将多级渐远层级 `i - 1` 转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。这个转换等待当前的 blit 指令完成。所有采样操作都将等待这个转换完成。

```rust,noplaypen
if mip_width > 1 {
    mip_width /= 2;
}

if mip_height > 1 {
    mip_height /= 2;
}
```

在循环体结束时，我们将当前的多级渐远维度除以 2。我们在除法之前检查每个维度，以确保该维度永远不会变为 0。这处理了图像不是正方形的情况，因为如果图像不是正方形，其中一个多级渐远维度会在另一个维度之前达到 1。当这种情况发生时，在剩下的层级中该维度应该保持 1。

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

在我们结束指令缓冲之前，我们插入了一个管线屏障。这个屏障将最后一个多级渐远层级从 `vk::ImageLayout::TRANSFER_DST_OPTIMAL` 转换为 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`。这在循环中没有处理，因为最后一个多级渐远层级不会被 blit 操作。

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

我们的纹理图像的多级渐远现在已经填充好了。

## 检查线性过滤支持

使用内置指令 `cmd_blit_image` 来生成所有的多级渐远层级非常方便，但不幸的是，并非所有的平台都保证支持它。`cmd_blit_image` 要求我们使用的纹理图像格式支持线性过滤，这可以通过 `get_physical_device_format_properties` 指令来检查。我们将在 `generate_mipmaps` 函数中添加一个检查。

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

`vk::FormatProperties` 结构有三个字段，分别命名为 `linear_tiling_features`、`optimal_tiling_features` 和 `buffer_features`，它们描述了格式在使用方式不同的情况下的使用方式。我们使用最佳平铺格式创建纹理图像，所以我们需要检查 `optimal_tiling_features`。可以使用 `vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR` 来检查线性过滤特性的支持。

在不支持的情况下有两种替代方案。你可以实现一个函数，搜索常见的纹理图像格式，找到一个支持线性 blit 的格式，或者你可以在你的软件中实现多级渐远的生成。然后，每个多级渐远层级可以以与加载原始图像相同的方式加载到图像中。

应该指出的是，在实践中，程序通常不会在运行时生成多级渐远层级。通常，它们是预先生成的，并与基础层级一起存储在纹理文件中，以提高加载速度。在软件中实现调整大小并从文件加载多个层级的功能留给读者作为练习。

## 采样器

`vk::Image` 中保存了多级渐远数据，而 `vk::Sampler` 控制了渲染时如何读取这些数据。Vulkan 允许我们指定 `min_lod`、`max_lod`、`mip_lod_bias` 和 `mipmap_mode`（"LOD" 意味着 "细节层级"）。当采样纹理时，采样器根据以下伪代码选择一个多级渐远层级：

```rust,noplaypen
// 物体越近就越小，可以为负值
let mut lod = get_lod_level_from_screen_size();

lod = clamp(lod + mip_lod_bias, min_lod, max_lod);

// 截断到纹理中多级渐远层级的数量
let level = clamp(floor(lod), 0, texture.mip_levels - 1);

let color = if mipmap_mode == vk::SamplerMipmapMode::NEAREST {
    sample(level)
} else {
    blend(sample(level), sample(level + 1))
};
```

如果 `sampler_info.mipmap_mode`（多级渐远模式）是 `vk::SamplerMipmapMode::NEAREST`，则 `lod` 会选择一个用于采样的多级渐远层级。如果多级渐远模式是 `vk::SamplerMipmapMode::LINEAR`，则 `lod` 用于选择要采样的两个多级渐远层级，对这些层级进行采样，并对结果线性混合。

采样操作也受 `lod` 的影响：

```rust,noplaypen
let color = if lod <= 0 {
    read_texture(uv, mag_filter)
} else {
    read_texture(uv, min_filter)
};
```

如果物体靠近相机，`mag_filter` 将被用作过滤器。如果物体离相机更远，`min_filter` 将被用作过滤器。通常，`lod` 是非负的，只有在靠近相机时才是 0。`mip_lod_bias` 让我们强制 Vulkan 使用比它通常使用的更低的 `lod` 和 `level`。

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

要使用所有的多级渐远层级，我们将 `min_lod` 设置为 `0.0`，`max_lod` 设置为多级渐远层级的数量。我们没有理由改变 `lod` 值，所以我们将 `mip_lod_bias` 设置为 0.0f。

现在运行你的程序，你应该会看到以下内容：

![](../images/mipmaps.png)

我们的场景太简单了，所以没有什么明显的区别。如果你仔细观察，你会发现微妙的区别（如果你在单独的标签中打开下面的图像，你会发现区别很容易被发现，这样你就可以看到它的全尺寸）。

![](../images/mipmaps_comparison.png)

最明显的区别之一是斧头。有了多级渐远，深灰色和浅灰色区域之间的边界已经被平滑了。要是没有多级渐远，这些边界要锐利得多。在这张图像中，斧头被放大了 8 倍，以显示有多级渐远和没有多级渐远（没有任何过滤，所以像素只是被扩展了）的情况。

![](../images/mipmaps_comparison_axe.png)

你可以玩一下采样器的设置，看看它们如何影响多级渐远。例如，通过改变 `min_lod`，你可以强制采样器不使用最低的多级渐远层级：

```rust,noplaypen
.min_lod(data.mip_levels as f32 / 2.0)
```

这些设置将产生这样的图像：

![](../images/highmipmaps.png)

这是当物体离相机更远时，更高的多级渐远层级将如何使用。
