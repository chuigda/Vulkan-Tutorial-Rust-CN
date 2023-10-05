# 图像

> 原文链接：<https://kylemayes.github.io/vulkanalia/texture/images.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/23_texture_image.rs)

我们已经使用了逐顶点颜色来为几何体上色，但这是一种相当受限的方法。在本章教程中，我们将实现纹理映射（texture mapping）来让几何体看上去更有趣。这也使得我们可以在后续的章节中加载并绘制出基础的 3D 模型。

向我们的应用添加纹理需要以下步骤：

* 创建一个设备内存中的图像对象
* 从图像文件向其中填充像素
* 创建一个图像采样器
* 添加一个组合图像采样器描述符，用来从纹理中采样颜色

之前我们已经处理过图像对象了，但那些是由交换链扩展自动创建的。这次我们将自己创建一个图像对象。创建一个图像并向其中填充数据的过程与创建顶点缓冲类似：我们先创建一个暂存缓冲并向它填充像素数据，然后再将这些数据复制到最终用来渲染的图像对象中。直接创建一个暂存图像也可以，但 Vulkan 允许直接从 `vk::Buffer` 复制像素到图像中，而且这样做实际上在某些硬件上[更快](https://developer.nvidia.com/vulkan-memory-management)。我们首先创建一个暂存缓冲并用像素值填充它，然后我们创建一个图像，并将像素复制到其中。创建图像与创建缓冲并没有太大的区别。它涉及到查询内存需求、分配设备内存并绑定，就像我们以前看到的那样。

然而，当使用图像时，我们还需要注意一些额外的事情。图像可以有不同的*布局*，布局会影响像素在内存中的组织方式。受限于图形硬件的工作方式，简单地按行存储像素可能无法带来最佳性能。当对图像执行任何操作时，你必须确保它们具有最适合在该操作中使用的布局。我们其实在指定渲染流程时已经见到了其中一些布局：

* `vk::ImageLayout::PRESENT_SRC_KHR` &ndash; 最适合用作呈现的布局&nbsp;
* `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL` &ndash; 最适合用作从片元着色器写入颜色的附件的布局&nbsp;
* `vk::ImageLayout::TRANSFER_SRC_OPTIMAL` &ndash; 最适合用作 `cmd_copy_image_to_buffer` 这类传输操作的数据源的布局&nbsp;
* `vk::ImageLayout::TRANSFER_DST_OPTIMAL` &ndash; 最适合用作 `cmd_copy_buffer_to_image` 这类传输操作的目标的布局&nbsp;
* `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL` &ndash; 最适合用于从一个着色器中采样的布局&nbsp;

转换图像布局最常见的方法之一是使用*管线屏障（pipeline barrier）*。管线屏障主要用于同步对资源的访问，例如确保图像在读取之前已经被写入。但管线屏障也可以用于转换布局。在本章中，我们将看到管线屏障是如何用于转换布局的。此外，当使用 `vk::SharingMode::EXCLUSIVE` 时，屏障还可以用于在队列族之间传递图像的所有权。

## 图像库

能用来加载图像的库有很多，你甚至可以自己编写代码来加载 BMP 和 PPM 等简单格式。在本教程中，我们将使用 [`png`](https://crates.io/crates/png) crate，你应该已经将它添加到程序的依赖中了。

## 加载图像

我们需要打开图像文件。添加以下导入：

```rust,noplaypen
use std::fs::File;
```

创建一个新函数 `create_texture_image`，我们将用它加载图像并将其上传到一个 Vulkan 图像对象中。我们要使用指令缓冲，所以它应该在 `create_command_pool` 之后调用。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_command_pool(&instance, &device, &mut data)?;
        create_texture_image(&instance, &device, &mut data)?;
        // ...
    }
}

unsafe fn create_texture_image(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

创建一个与 `shaders` 目录同一级的新目录 `resources` 用来存放纹理图像。我们将从该目录加载一个名为 `texture.png` 的图像。我选择使用下面这张 [以 CC0 协议发布的图像](https://pixabay.com/en/statue-sculpture-fig-historically-1275469/)，并将其调整到 512 x 512 像素大小，你也可以随意选择任何你想使用的（带有 alpha 通道的）PNG 图像。

![](../images/texture.png)

使用这个库加载图像非常简单：

```rust,noplaypen
unsafe fn create_texture_image(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let image = File::open("resources/texture.png")?;

    let decoder = png::Decoder::new(image);
    let mut reader = decoder.read_info()?;

    let mut pixels = vec![0;  reader.info().raw_bytes()];
    reader.next_frame(&mut pixels)?;

    let size = reader.info().raw_bytes() as u64;
    let (width, height) = reader.info().size();

    Ok(())
}
```

这段代码将用每像素 4 个字节的数据填充 `pixels` 列表，总共将会有 `width * height * 4` 个值。注意 `png` crate [目前还不支持将 RGB 图像转换为 RGBA 图像](https://github.com/image-rs/image-png/issues/239)，并且后续代码将假设像素数据拥有 alpha 通道。因此，你需要确保使用带有 alpha 通道的 PNG 图像（例如上面的图像）。

## 暂存缓冲

现在我们将在主机可见的内存中创建一个缓冲，以便我们使用 `map_memory` 并将像素复制到其中。缓冲应该在主机可见的内存中，这样我们可以将其映射。它还应该能被用作传输源，以便我们稍后能将其复制到图像中：

```rust,noplaypen
let (staging_buffer, staging_buffer_memory) = create_buffer(
    instance,
    device,
    data,
    size,
    vk::BufferUsageFlags::TRANSFER_SRC,
    vk::MemoryPropertyFlags::HOST_COHERENT | vk::MemoryPropertyFlags::HOST_VISIBLE,
)?;
```

然后，我们可以直接将从图像加载库中获取的像素值复制到缓冲中：

```rust,noplaypen
let memory = device.map_memory(
    staging_buffer_memory,
    0,
    size,
    vk::MemoryMapFlags::empty(),
)?;

memcpy(pixels.as_ptr(), memory.cast(), pixels.len());

device.unmap_memory(staging_buffer_memory);
```

## 纹理图像

尽管我们可以让着色器直接访问缓冲中的像素值，但最好还是使用 Vulkan 中的图像对象来实现这一目标。使用图像对象，我们能够使用 2D 坐标更轻松、更快速地检索颜色。图像对象中的像素称为纹素（texel），我们从现在开始使用这个名字。将下列新的字段添加到 `AppData` 中：

```rust,noplaypen
struct AppData {
    // ...
    texture_image: vk::Image,
    texture_image_memory: vk::DeviceMemory,
}
```

图像的参数在 `vk::ImageCreateInfo` 结构体中指定：

```rust,noplaypen
let info = vk::ImageCreateInfo::builder()
    .image_type(vk::ImageType::_2D)
    .extent(vk::Extent3D { width, height, depth: 1 })
    .mip_levels(1)
    .array_layers(1)
    // continued...
```

在 `image_type` 字段中指定的图像类型会告诉 Vulkan 图像中的纹素将使用什么样的坐标系来寻址。我们可以创建 1D、2D 和 3D 图像。一维图像可以用来存储数据或渐变；二维图像主要用于纹理；三维图像可以用来存储体素（voxel）体积。`extent` 字段指定了图像的尺寸，也就是每个坐标轴上有多少个纹素。这就是为什么 `depth` 必须是 `1` 而不是 `0`。我们的纹理不会是一个数组，并且我们现在也不会使用多级渐远。

```rust,noplaypen
    .format(vk::Format::R8G8B8A8_SRGB)
```

Vulkan 支持许多图像格式，但我们应该为纹素使用与缓冲中的像素相同的格式，否则复制操作将会失败。

```rust,noplaypen
    .tiling(vk::ImageTiling::OPTIMAL)
```

`tiling` 字段可以从以下两个值中选择：

* `vk::ImageTiling::LINEAR` &ndash; 纹素按行主序排列，和我们的 `pixels` 数组一样
* `vk::ImageTiling::OPTIMAL` &ndash; 纹素按实现定义的最佳访问顺序排列

与图像的布局不同，平铺（tiling）模式无法在之后更改。如果你想要能直接访问图像内存中的纹素，那么你必须使用 `vk::ImageTiling::LINEAR`。我们不需要这样，因为我们会使用暂存缓冲而不是暂存图像。我们将使用 `vk::ImageTiling::OPTIMAL` 来实现着色器的高效访问。

```rust,noplaypen
    .initial_layout(vk::ImageLayout::UNDEFINED)
```

图像的 `initial_layout` 只有两个可能的值：

* `vk::ImageLayout::UNDEFINED` &ndash; 不可被 GPU 使用，第一次转换将会丢弃纹素。
* `vk::ImageLayout::PREINITIALIZED` &ndash; 不可被 GPU 使用，但第一次转换将会保留纹素。

在少数情况下，第一次转换时需要保留纹素。例如，如果你将某个具有 `vk::ImageTiling::LINEAR` 平铺模式的图像用作暂存图像，你需要将纹素数据上传到其中，然后将图像转换为传输源 —— 这个过程中不能丢失数据。然而，在我们的例子中，我们是图像转换为传输目标，然后再从缓冲对象中将纹素数据复制到其中，所以我们不需要这个特性，可以安全地使用 `vk::ImageLayout::UNDEFINED`。

```rust,noplaypen
    .usage(vk::ImageUsageFlags::SAMPLED | vk::ImageUsageFlags::TRANSFER_DST)
```

`usage` 字段的语义与缓冲创建时的语义相同。图像将被用作缓冲复制的目标，因此它应该被设置为传输目标。我们还希望能够从着色器中访问图像来给网格上色，所以 `usage` 也应该需要 `vk::ImageUsageFlags::SAMPLED`。

```rust,noplaypen
    .sharing_mode(vk::SharingMode::EXCLUSIVE)
```

图像只会被一个队列族使用，即支持图形操作（因此也支持传输）的队列族。

```rust,noplaypen
    .samples(vk::SampleCountFlags::_1)
```

`samples` 标志与多重采样有关。这只与用作附件的图像有关，所以我们只需要一个采样。

```rust,noplaypen
    .flags(vk::ImageCreateFlags::empty()); // Optional.
```

在图像上还有一些可选的标志，允许控制诸如稀疏图像（sparse image）之类的更高级属性。稀疏图像是只有某些区域储存在内存中的图像。例如，如果你使用一个 3D 纹理来存储体素地形，那么你可以使用稀疏图像来避免分配内存来存储大量的“空气”值。我们在本教程中不会使用它，所以你可以省略这个字段的生成器方法，这将把它设置为默认值（一个空的标志集）。

```rust,noplaypen
data.texture_image = device.create_image(&info, None)?;
```

图像使用 `create_image` 创建，没有任何特别值得注意的参数。图形硬件可能不支持 `vk::Format::R8G8B8A8_SRGB` 格式，这时你需要有一系列可用的替代格式，并选择其中能被支持的最好的一个。然而，这个格式的支持十分广泛，我们将跳过这一步。使用不同的格式也需要烦人的转换。我们将在深度缓冲章节中回过头来实现这样的一个系统。

```rust,noplaypen
let requirements = device.get_image_memory_requirements(data.texture_image);

let info = vk::MemoryAllocateInfo::builder()
    .allocation_size(requirements.size)
    .memory_type_index(get_memory_type_index(
        instance,
        data,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
        requirements,
    )?);

data.texture_image_memory = device.allocate_memory(&info, None)?;

device.bind_image_memory(data.texture_image, data.texture_image_memory, 0)?;
```

给图像分配内存的方式与给缓冲分配内存的方式完全相同，除了需要使用 `get_image_memory_requirements` 而不是 `get_buffer_memory_requirements`，使用 `bind_image_memory` 而不是 `bind_buffer_memory`。

这个函数已经变得相当长了，并且在后面的章节中还需要创建更多的图像，所以我们应该为图像创建抽象出一个 `create_image` 函数，和缓冲一样。创建这个函数并将图像对象的创建和内存分配移入其中：

```rust,noplaypen
unsafe fn create_image(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    width: u32,
    height: u32,
    format: vk::Format,
    tiling: vk::ImageTiling,
    usage: vk::ImageUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> Result<(vk::Image, vk::DeviceMemory)> {
    let info = vk::ImageCreateInfo::builder()
        .image_type(vk::ImageType::_2D)
        .extent(vk::Extent3D {
            width,
            height,
            depth: 1,
        })
        .mip_levels(1)
        .array_layers(1)
        .format(format)
        .tiling(tiling)
        .initial_layout(vk::ImageLayout::UNDEFINED)
        .usage(usage)
        .samples(vk::SampleCountFlags::_1)
        .sharing_mode(vk::SharingMode::EXCLUSIVE);

    let image = device.create_image(&info, None)?;

    let requirements = device.get_image_memory_requirements(image);

    let info = vk::MemoryAllocateInfo::builder()
        .allocation_size(requirements.size)
        .memory_type_index(get_memory_type_index(
            instance,
            data,
            properties,
            requirements,
        )?);

    let image_memory = device.allocate_memory(&info, None)?;

    device.bind_image_memory(image, image_memory, 0)?;

    Ok((image, image_memory))
}
```

我把宽度、高度、格式、平铺模式、用途和内存属性变成了参数，因为这些都将在不同图像之间变化。

`create_texture_image` 函数现在可以简化为：

```rust,noplaypen
unsafe fn create_texture_image(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let image = File::open("resources/texture.png")?;

    let decoder = png::Decoder::new(image);
    let mut reader = decoder.read_info()?;

    let mut pixels = vec![0;  reader.info().raw_bytes()];
    reader.next_frame(&mut pixels)?;

    let size = reader.info().raw_bytes() as u64;
    let (width, height) = reader.info().size();

    let (staging_buffer, staging_buffer_memory) = create_buffer(
        instance,
        device,
        data,
        size,
        vk::BufferUsageFlags::TRANSFER_SRC,
        vk::MemoryPropertyFlags::HOST_COHERENT | vk::MemoryPropertyFlags::HOST_VISIBLE,
    )?;

    let memory = device.map_memory(
        staging_buffer_memory,
        0,
        size,
        vk::MemoryMapFlags::empty(),
    )?;

    memcpy(pixels.as_ptr(), memory.cast(), pixels.len());

    device.unmap_memory(staging_buffer_memory);

    let (texture_image, texture_image_memory) = create_image(
        instance,
        device,
        data,
        width,
        height,
        vk::Format::R8G8B8A8_SRGB,
        vk::ImageTiling::OPTIMAL,
        vk::ImageUsageFlags::SAMPLED | vk::ImageUsageFlags::TRANSFER_DST,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    )?;

    data.texture_image = texture_image;
    data.texture_image_memory = texture_image_memory;

    Ok(())
}
```

## 布局转换

我们现在要写的函数再次涉及到记录和执行指令缓冲，所以现在是将这些逻辑移入几个辅助函数的好时机：

```rust,noplaypen
unsafe fn begin_single_time_commands(
    device: &Device,
    data: &AppData,
) -> Result<vk::CommandBuffer> {
    let info = vk::CommandBufferAllocateInfo::builder()
        .level(vk::CommandBufferLevel::PRIMARY)
        .command_pool(data.command_pool)
        .command_buffer_count(1);

    let command_buffer = device.allocate_command_buffers(&info)?[0];

    let info = vk::CommandBufferBeginInfo::builder()
        .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);

    device.begin_command_buffer(command_buffer, &info)?;

    Ok(command_buffer)
}

unsafe fn end_single_time_commands(
    device: &Device,
    data: &AppData,
    command_buffer: vk::CommandBuffer,
) -> Result<()> {
    device.end_command_buffer(command_buffer)?;

    let command_buffers = &[command_buffer];
    let info = vk::SubmitInfo::builder()
        .command_buffers(command_buffers);

    device.queue_submit(data.graphics_queue, &[info], vk::Fence::null())?;
    device.queue_wait_idle(data.graphics_queue)?;

    device.free_command_buffers(data.command_pool, &[command_buffer]);

    Ok(())
}
```

这些函数抽取自 `copy_buffer` 中原有的代码。你现在可以将 `copy_buffer` 函数简化为：

```rust,noplaypen
unsafe fn copy_buffer(
    device: &Device,
    data: &AppData,
    source: vk::Buffer,
    destination: vk::Buffer,
    size: vk::DeviceSize,
) -> Result<()> {
    let command_buffer = begin_single_time_commands(device, data)?;

    let regions = vk::BufferCopy::builder().size(size);
    device.cmd_copy_buffer(command_buffer, source, destination, &[regions]);

    end_single_time_commands(device, data, command_buffer)?;

    Ok(())
}
```

现在我们要调用函数 `cmd_copy_buffer_to_image` 来记录将像素从暂存缓冲复制到图像中的指令，但这个指令首先要求图像处于正确的布局。创建一个新函数来处理布局转换：

```rust,noplaypen
unsafe fn transition_image_layout(
    device: &Device,
    data: &AppData,
    image: vk::Image,
    format: vk::Format,
    old_layout: vk::ImageLayout,
    new_layout: vk::ImageLayout,
) -> Result<()> {
    let command_buffer = begin_single_time_commands(device, data)?;

    end_single_time_commands(device, data, command_buffer)?;

    Ok(())
}
```

转换布局最常用的方法之一是使用*图像内存屏障（image memory barrier）*。这样的管线屏障通常用于同步对资源的访问，例如确保对缓冲的写入在读取之前完成，但它也可以用于转换图像布局，以及在使用 `vk::SharingMode::EXCLUSIVE` 时在队列族之间传递图像对象的所有权。对于缓冲，有一个等效的*缓冲内存屏障（buffer memory barrier）*。

```rust,noplaypen
let barrier = vk::ImageMemoryBarrier::builder()
    .old_layout(old_layout)
    .new_layout(new_layout)
    // continued...
```

前两个字段指定了布局转换。如果你不关心图像中现有的内容，那么可以使用 `vk::ImageLayout::UNDEFINED` 作为 `old_layout`。

```rust,noplaypen
    .src_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
    .dst_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
```

如果你使用屏障来在队列族之间传输图像对象的所有权，那么这两个字段应该是队列族的索引。如果你不想这样做，那么它们必须被显式设置为 `vk::QUEUE_FAMILY_IGNORED`（不是默认值！）。

```rust,noplaypen
    .image(image)
    .subresource_range(subresource)
```

`image` 和 `subresource_range` 指定了受影响的图像和图像中的特定部分。我们需要在定义图像内存屏障之前定义 `subresource`：

```rust,noplaypen
let subresource = vk::ImageSubresourceRange::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .base_mip_level(0)
    .level_count(1)
    .base_array_layer(0)
    .layer_count(1);
```

我们的图像不是一个数组，也没有多级渐远层级，所以只指定了一个多级渐远层级和数组层。

```rust,noplaypen
    .src_access_mask(vk::AccessFlags::empty())  // TODO
    .dst_access_mask(vk::AccessFlags::empty()); // TODO
```

屏障主要用于同步，所以你必须指定哪些涉及资源的操作类型需要在屏障之前发生，以及哪些涉及资源的操作需要在屏障处等待。尽管我们已经使用 `queue_wait_idle` 来手动同步，但我们仍然需要这样做。这些值取决于旧布局和新布局，所以我们会在弄清楚要使用哪些转换之后再回到这里。

```rust,noplaypen
device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::empty(), // TODO
    vk::PipelineStageFlags::empty(), // TODO
    vk::DependencyFlags::empty(),
    &[] as &[vk::MemoryBarrier],
    &[] as &[vk::BufferMemoryBarrier],
    &[barrier],
);
```

所有类型的管线屏障都使用相同的函数提交。指令缓冲之后的第一个参数指定了在屏障之前应该发生的操作所在的管线阶段。第二个参数指定了在屏障处等待的操作所在的管线阶段。在屏障之前和之后允许指定的管线阶段取决于在屏障之前和之后如何使用资源。允许的值在 Vulkan 规范的[这个表格](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#synchronization-access-types-supported)中列出。例如，如果你想在屏障之后读取一个 uniform，你应该指定一个 `vk::AccessFlags::UNIFORM_READ` 的用途和最早从 uniform 中读取的着色器作为管线阶段，比如 `vk::PipelineStageFlags::FRAGMENT_SHADER`。对于这类用途，指定非着色器管线阶段是没有意义的。校验层会在你指定与用途类型不匹配的管线阶段时发出警告。

第四个参数是一个空的 `vk::DependencyFlags` 集合或 `vk::DependencyFlags::BY_REGION`。后者将屏障变成每个区域的条件。举例来说，这意味着你可以从在此之前已经写入的资源的部分开始读取。

最后三个参数引用了三种可用类型的管线屏障的切片：内存屏障、缓冲内存屏障以及我们这里使用的图像内存屏障。注意，我们目前还没有使用 `vk::Format` 参数，但我们将在深度缓冲章节中使用它来进行特殊的转换。

## 复制缓冲至图像

在我们回到 `create_texture_image` 之前，我们还要写另一个辅助函数 `copy_buffer_to_image`：

```rust,noplaypen
unsafe fn copy_buffer_to_image(
    device: &Device,
    data: &AppData,
    buffer: vk::Buffer,
    image: vk::Image,
    width: u32,
    height: u32,
) -> Result<()> {
    let command_buffer = begin_single_time_commands(device, data)?;

    end_single_time_commands(device, data, command_buffer)?;

    Ok(())
}
```

正如缓冲复制一样，你需要指定缓冲的哪一部分将被复制到图像的哪一部分。这是通过 `vk::BufferImageCopy` 结构体来完成的：

```rust,noplaypen
let subresource = vk::ImageSubresourceLayers::builder()
    .aspect_mask(vk::ImageAspectFlags::COLOR)
    .mip_level(0)
    .base_array_layer(0)
    .layer_count(1);

let region = vk::BufferImageCopy::builder()
    .buffer_offset(0)
    .buffer_row_length(0)
    .buffer_image_height(0)
    .image_subresource(subresource)
    .image_offset(vk::Offset3D { x: 0, y: 0, z: 0 })
    .image_extent(vk::Extent3D { width, height, depth: 1 });
```

这里的大部分字段无需解释。`buffer_offset` 指定了缓冲中像素值开始的字节偏移量。`buffer_row_length` 和 `buffer_image_height` 字段指定了像素在内存中的布局。例如，你可以在图像的行之间有一些填充字节。对于这两个字段都指定 `0` 表示像素就像我们现在这样紧密地排列。`image_subresource`、`image_offset` 和 `image_extent` 字段指示我们要将像素复制到图像的哪个部分。

使用 `cmd_copy_buffer_to_image` 函数将从缓冲到图像的复制操作加入队列：

```rust,noplaypen
device.cmd_copy_buffer_to_image(
    command_buffer,
    buffer,
    image,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    &[region],
);
```

第四个参数表示图像当前使用哪一种布局。我在这里假设图像已经转换为最适合复制像素的布局。现在我们只复制一段像素到整个图像，但也可以指定一个 `vk::BufferImageCopy` 数组来在一次操作中将这个缓冲的多个不同拷贝复制到图像中。

## 准备纹理图像

我们现在已经拥有了完成纹理图像所需的所有工具，所以我们要回到 `create_texture_image` 函数。我们先前在那里做的最后一件事是创建纹理图像。下一步是将暂存缓冲复制到纹理图像。这涉及到两个步骤：

* 转换纹理图像为 `vk::ImageLayout::TRANSFER_DST_OPTIMAL`
* 执行从缓冲到图像的复制操作

这可以用我们刚刚创建的函数轻易完成：

```rust,noplaypen
transition_image_layout(
    device,
    data,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::UNDEFINED,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
)?;

copy_buffer_to_image(
    device,
    data,
    staging_buffer,
    data.texture_image,
    width,
    height,
)?;
```

图像是使用 `vk::ImageLayout::UNDEFINED` 布局创建的，所以在转换 `texture_image` 时将它指定为旧布局。请记住，我们可以这样做是因为在执行复制操作之前我们不关心它的内容。

为了能够开始从着色器中采样纹理图像，我们需要最后一个转换以供着色器访问：

```rust,noplaypen
transition_image_layout(
    device,
    data,
    data.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
)?;
```

## 转换屏障掩码

如果你现在启用校验层并运行你的应用程序，那么你会看到它警告 `transition_image_layout` 中的访问掩码和管线阶段无效。我们仍然需要根据转换中的布局来设置它们。

我们需要处理两个转换：

* 未定义 → 传输目标 &ndash; 不需要等待任何东西的传输写入
* 传输目标 → 着色器读取 &ndash; 着色器读取应该等待传输写入，特别是片段着色器中的着色器读取，因为我们会在那里使用纹理

这些规则可以使用以下访问掩码和管线阶段来指定，它们应该被加在 `transition_image_layout` 的开头：

```rust,noplaypen
let (
    src_access_mask,
    dst_access_mask,
    src_stage_mask,
    dst_stage_mask,
) = match (old_layout, new_layout) {
    (vk::ImageLayout::UNDEFINED, vk::ImageLayout::TRANSFER_DST_OPTIMAL) => (
        vk::AccessFlags::empty(),
        vk::AccessFlags::TRANSFER_WRITE,
        vk::PipelineStageFlags::TOP_OF_PIPE,
        vk::PipelineStageFlags::TRANSFER,
    ),
    (vk::ImageLayout::TRANSFER_DST_OPTIMAL, vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL) => (
        vk::AccessFlags::TRANSFER_WRITE,
        vk::AccessFlags::SHADER_READ,
        vk::PipelineStageFlags::TRANSFER,
        vk::PipelineStageFlags::FRAGMENT_SHADER,
    ),
    _ => return Err(anyhow!("Unsupported image layout transition!")),
};
```

然后使用访问标志和管线阶段掩码更新 `vk::ImageMemoryBarrier` 结构体和 `cmd_pipeline_barrier` 调用：

```rust,noplaypen
let barrier = vk::ImageMemoryBarrier::builder()
    .old_layout(old_layout)
    .new_layout(new_layout)
    .src_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
    .dst_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
    .image(image)
    .subresource_range(subresource)
    .src_access_mask(src_access_mask)
    .dst_access_mask(dst_access_mask);

device.cmd_pipeline_barrier(
    command_buffer,
    src_stage_mask,
    dst_stage_mask,
    vk::DependencyFlags::empty(),
    &[] as &[vk::MemoryBarrier],
    &[] as &[vk::BufferMemoryBarrier],
    &[barrier],
);
```

如你在之前提到的列表中所见，传输写入必须发生在管线传输阶段。由于写入不需要任何等待，你可以为屏障前操作指定空访问掩码与最早的管线阶段 `vk::PipelineStageFlags::TOP_OF_PIPE`。注意，`vk::PipelineStageFlags::TRANSFER` 不是一个在图形与计算管线中的*真实*阶段。它更像是一个传输操作发生的伪阶段。欲知更多详情与其它伪阶段的例子，请见[文档](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkPipelineStageFlagBits.html)。

图像将在相同的管线阶段被写入，然后在片段着色器中被读取。这就是为什么我们在片段着色器管线阶段指定了着色器读取访问。

如果我们将来需要做更多的转换，那么我们会扩展这个函数。现在，程序应该可以成功运行。当然，现在还不会有视觉上的变化。

有一件事需要注意，那就是提交指令缓冲在一开始时会导致隐式的 `vk::AccessFlags::HOST_WRITE` 同步。由于 `transition_image_layout` 函数执行的指令缓冲只有一个指令，如果你在布局转换中需要一个 `vk::AccessFlags::HOST_WRITE` 依赖，那么你可以使用这个隐式同步并将 `src_access_mask` 设置为 `vk::AccessFlags::empty()`。你可以选择是否要明确地指定它，但我个人不喜欢依赖这些类似 OpenGL 的“隐藏”操作。

实际上有一种特殊的图像布局类型 `vk::ImageLayout::GENERAL`，它支持所有操作。当然，它的问题也就是它不一定能为任何操作提供最佳性能。对于一些特殊情况它是必需的，比如将图像同时用作输入和输出，或者在它离开预初始化布局后读取图像。

到目前为止，所有提交指令的辅助函数都被设置为同步执行，因为它们会等待队列空闲。对于实际应用程序，建议将这些操作组合到一个单独的指令缓冲中，并异步执行以获得更高的吞吐量，对于在 `create_texture_image` 函数中的转换和复制更是如此。试试看通过创建 `setup_command_buffer` 来让辅助函数记录指令，并添加一个 `flush_setup_commands` 来执行到目前为止已经记录的指令。最好在纹理映射能正常运行之后这样做，以检查纹理资源是否仍然配置正确。

## 善后

在 `create_texture_image` 函数的最后将暂存缓冲及其内存清理掉：

```rust,noplaypen
device.destroy_buffer(staging_buffer, None);
device.free_memory(staging_buffer_memory, None);
```

主要纹理图像将会一直使用到程序结束：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_image(self.data.texture_image, None);
    self.device.free_memory(self.data.texture_image_memory, None);
    // ...
}
```

图像现在包含了纹理，但我们仍然需要一种方法来从图形管线中访问它。我们将在下一章中讨论这个问题。
