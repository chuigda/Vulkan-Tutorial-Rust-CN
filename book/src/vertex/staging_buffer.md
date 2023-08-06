# 暂存缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/vertex/staging_buffer.html>
>
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/19_staging_buffer.rs)

目前我们的顶点缓冲可以正常工作，但是能直接从 CPU 访问的内存对于从显卡读取而言可能并不是最优的。最优内存具有 `vk::MemoryPropertyFlags::DEVICE_LOCAL` 标志，通常位于独立显卡上，无法由 CPU 访问。在本章中，我们将创建两个顶点缓冲。首先是位于 CPU 可访问内存中的*暂存缓冲*，用于将顶点数组中的数据上传至其中；然后是位于设备本地内存中的最终顶点缓冲。接着，我们将使用缓冲复制指令将数据从暂存缓冲复制到实际的顶点缓冲中。

## 传输队列

缓冲复制指令需要一个支持传输操作的队列族，这种队列族具有 `vk::QueueFlags::TRANSFER` 标志。好消息是，任何具有 `vk::QueueFlags::GRAPHICS` 或 `vk::QueueFlags::COMPUTE` 能力的队列族已经隐式地支持 `vk::QueueFlags::TRANSFER` 操作。在这种情况下，实现不需要在 `queue_flags` 中显式列出这个标志。

如果你愿意接受挑战，你仍然可以尝试为传输操作使用不同的队列族。这将需要你对程序进行以下修改：

- 修改 `QueueFamilyIndices` 和 `QueueFamilyIndices::get`，以明确寻找具有 `vk::QueueFlags::TRANSFER` 标志但不具有 `vk::QueueFlags::GRAPHICS` 的队列族。
- 修改 `create_logical_device`，以请求传输队列的句柄。
- 为在传输队列族上提交的指令缓冲创建第二个指令池。
- 将资源的 `sharing_mode` 改为 `vk::SharingMode::CONCURRENT`，并指定图形队列族和传输队列族。
- 将任何传输指令（在本章中将使用的 `cmd_copy_buffer` 等）提交到传输队列，而不是图形队列。

虽然需要付出一些努力，但这将让你深入了解在不同队列族之间共享资源的重要知识。

## 抽象化缓冲创建

由于我们将在本章中创建多个缓冲，将缓冲创建操作移动到一个辅助函数中是个不错的主意。创建一个名为 `create_buffer` 的新函数，并将 `create_vertex_buffer` 中的代码（除了映射部分）迁移到该函数中：

```rust,noplaypen
unsafe fn create_buffer(
    instance: &Instance,
    device: &Device,
    data: &AppData,
    size: vk::DeviceSize,
    usage: vk::BufferUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> Result<(vk::Buffer, vk::DeviceMemory)> {
    let buffer_info = vk::BufferCreateInfo::builder()
        .size(size)
        .usage(usage)
        .sharing_mode(vk::SharingMode::EXCLUSIVE);

    let buffer = device.create_buffer(&buffer_info, None)?;

    let requirements = device.get_buffer_memory_requirements(buffer);

    let memory_info = vk::MemoryAllocateInfo::builder()
        .allocation_size(requirements.size)
        .memory_type_index(get_memory_type_index(
            instance,
            data,
            properties,
            requirements,
        )?);

    let buffer_memory = device.allocate_memory(&memory_info, None)?;

    device.bind_buffer_memory(buffer, buffer_memory, 0)?;

    Ok((buffer, buffer_memory))
}
```

确保将缓冲大小、用法以及内存属性添加到函数参数，以便于我们使用此函数创建多种不同类型的缓冲。

现在，你可以从 `create_vertex_buffer` 中删除创建缓冲和分配内存的代码，改为调用 `create_buffer`：

```rust,noplaypen
unsafe fn create_vertex_buffer(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let size = (size_of::<Vertex>() * VERTICES.len()) as u64;

    let (vertex_buffer, vertex_buffer_memory) = create_buffer(
        instance,
        device,
        data,
        size,
        vk::BufferUsageFlags::VERTEX_BUFFER,
        vk::MemoryPropertyFlags::HOST_COHERENT | vk::MemoryPropertyFlags::HOST_VISIBLE,
    )?;

    data.vertex_buffer = vertex_buffer;
    data.vertex_buffer_memory = vertex_buffer_memory;

    let memory = device.map_memory(
        vertex_buffer_memory,
        0,
        size,
        vk::MemoryMapFlags::empty(),
    )?;

    memcpy(VERTICES.as_ptr(), memory.cast(), VERTICES.len());

    device.unmap_memory(vertex_buffer_memory);

    Ok(())
}
```

运行程序，确保顶点缓冲仍然正常工作。

## 使用暂存缓冲

现在，我们要修改 `create_vertex_buffer`，使其只将主机可见的缓冲作为临时缓冲，并将一个设备本地缓冲用作实际的顶点缓冲。

```rust,noplaypen
unsafe fn create_vertex_buffer(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let size = (size_of::<Vertex>() * VERTICES.len()) as u64;

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

    memcpy(VERTICES.as_ptr(), memory.cast(), VERTICES.len());

    device.unmap_memory(staging_buffer_memory);

    let (vertex_buffer, vertex_buffer_memory) = create_buffer(
        instance,
        device,
        data,
        size,
        vk::BufferUsageFlags::TRANSFER_DST | vk::BufferUsageFlags::VERTEX_BUFFER,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    )?;

    data.vertex_buffer = vertex_buffer;
    data.vertex_buffer_memory = vertex_buffer_memory;

    Ok(())
}
```

我们现在使用新的 `staging_buffer` 和 `staging_buffer_memory` 来映射和复制顶点数据。在本章中，我们将使用两个新的缓冲用法标志：

* `vk::BufferUsageFlags::TRANSFER_SRC` &ndash; 缓冲可以作为内存传输操作的源。
* `vk::BufferUsageFlags::TRANSFER_DST` &ndash; 缓冲可以作为内存传输操作的目标。

`vertex_buffer` 现在是从设备本地内存类型分配的，这通常意味着我们不能使用 `map_memory`。然而，我们可以将数据从 `staging_buffer` 复制到 `vertex_buffer`。我们必须为 `staging_buffer` 指定传输源标志，为 `vertex_buffer` 指定传输目标标志和顶点缓冲用法标志，来表明我们的意图。

接下来，我们将编写一个名为 `copy_buffer` 的函数，用于将内容从一个缓冲复制到另一个缓冲。

```rust,noplaypen
unsafe fn copy_buffer(
    device: &Device,
    data: &AppData,
    source: vk::Buffer,
    destination: vk::Buffer,
    size: vk::DeviceSize,
) -> Result<()> {
    Ok(())
}
```

内存传输操作与绘制指令一样，都需要通过指令缓冲来执行。因此，我们首先需要分配一个临时的指令缓冲。你可能希望为这些短暂的缓冲创建一个独立的指令池，因为实现可以对内存分配进行优化。在这种情况下，你应该在生成指令池时使用 `vk::CommandPoolCreateFlags::TRANSIENT` 标志。

```rust,noplaypen
unsafe fn copy_buffer(
    device: &Device,
    data: &AppData,
    source: vk::Buffer,
    destination: vk::Buffer,
    size: vk::DeviceSize,
) -> Result<()> {
    let info = vk::CommandBufferAllocateInfo::builder()
        .level(vk::CommandBufferLevel::PRIMARY)
        .command_pool(data.command_pool)
        .command_buffer_count(1);

    let command_buffer = device.allocate_command_buffers(&info)?[0];

    Ok(())
}
```

然后开始记录指令缓冲：

```rust,noplaypen
let info = vk::CommandBufferBeginInfo::builder()
    .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);

device.begin_command_buffer(command_buffer, &info)?;
```

我们将只使用这个指令缓冲一次，并在复制操作完成之前等待函数返回。使用 `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` 标志可以向驱动程序表明我们的意图，这是一个很好的实践。

```rust,noplaypen
let regions = vk::BufferCopy::builder().size(size);
device.cmd_copy_buffer(command_buffer, source, destination, &[regions]);
```

缓冲的内容通过 `cmd_copy_buffer` 指令进行传输。该指令以源缓冲、目标缓冲和待复制区域的数组为参数。区域由 `vk::BufferCopy` 结构体定义，结构体中包括源缓冲偏移量、目标缓冲偏移量和大小。需要注意的是，与 `map_memory` 指令不同，这里不能指定 `vk::WHOLE_SIZE`。

```rust,noplaypen
device.end_command_buffer(command_buffer)?;
```

这个指令缓冲仅包含复制指令，因此我们在复制指令之后停止记录。现在执行该指令缓冲以完成传输操作：

```rust,noplaypen
let command_buffers = &[command_buffer];
let info = vk::SubmitInfo::builder()
    .command_buffers(command_buffers);

device.queue_submit(data.graphics_queue, &[info], vk::Fence::null())?;
device.queue_wait_idle(data.graphics_queue)?;
```

与绘制指令不同，这次我们无需等待事件，而是立即在缓冲上执行传输操作。同样，有两种方法可以等待传输完成。我们可以使用围栏（fence），并使用 `wait_for_fences` 来等待，或者只需使用 `queue_wait_idle` 等待传输队列变为空闲状态。使用围栏可以让你同时安排多个传输并等待它们全部完成，而不必逐个执行。这可以给驱动程序更多优化的机会。

```rust,noplaypen
device.free_command_buffers(data.command_pool, &[command_buffer]);
```

别忘记清理用于传输操作的指令缓冲。

现在，我们可以在 `create_vertex_buffer` 函数中调用 `copy_buffer`，将顶点数据复制到设备本地缓冲：

```rust,noplaypen
copy_buffer(device, data, staging_buffer, vertex_buffer, size)?;
```

在从暂存缓冲复制数据到设备缓冲之后，不要忘记进行清理：

```rust,noplaypen
device.destroy_buffer(staging_buffer, None);
device.free_memory(staging_buffer_memory, None);
```

运行程序以验证你是否能再次看到熟悉的三角形。现在，顶点数据是从高性能内存加载的，尽管目前可能看不到改进。当我们开始渲染更复杂的几何图形时，这一点将变得更加重要。

## 结论

值得注意的是，在实际的应用程序中，你不应该为每个缓冲都调用 `allocate_memory`。内存分配的最大数量受到物理设备的 `max_memory_allocation_count` 限制，即使在高端硬件（如 NVIDIA GTX 1080）上，这个限制也可能低至 `4096`。要在同一时刻为大量对象分配内存，正确的方法是创建一个自定义的分配器，通过使用我们在许多函数中看到的 `offset` 参数，将单个分配分割为多个不同的对象。

然而，在本教程中可以为每个资源单独分配，因为目前我们不会接近这些限制。
