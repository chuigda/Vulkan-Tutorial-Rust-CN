# 顶点缓冲创建

> 原文链接：<https://kylemayes.github.io/vulkanalia/vertex/vertex_buffer_creation.html>
>
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/18_vertex_buffer.rs)

在Vulkan中，缓冲是用于存储可被图形卡读取的任意数据的内存区域。它们可以用于存储顶点数据，这正是我们将在本章中所做的，但它们也可以用于许多其他目的，这些将在以后的章节中探讨。与我们到目前为止处理过的Vulkan对象不同，缓冲不会自动为自己分配内存。前面章节中的工作已经表明，Vulkan API将程序员置于几乎所有事物的控制下，内存管理就是其中之一。

## 缓冲创建

首先，我们创建一个名为 `create_vertex_buffer` 的新函数，并在 `App::create` 函数中，在 `create_command_buffers` 之前调用它。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_vertex_buffer(&instance, &device, &mut data)?;
        create_command_buffers(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_vertex_buffer(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

创建缓冲需要填充一个 `vk::BufferCreateInfo` 结构体。

```rust,noplaypen
let buffer_info = vk::BufferCreateInfo::builder()
    .size((size_of::<Vertex>() * VERTICES.len()) as u64)
    // continued...
```

结构体的第一个字段是 `size` ，它指定缓冲的大小，以字节为单位。使用 `size_of` 可以很容易地计算出顶点数据的字节大小。

```rust,noplaypen
    .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
```

结构体的第二个字段是 `usage`，它表示缓冲中的数据将用于哪些目的。可以使用按位或指定多个目的。我们的用例是用于顶点缓冲，关于其他类型的用法将在以后的章节中讨论。

```rust,noplaypen
    .sharing_mode(vk::SharingMode::EXCLUSIVE);
```

和交换链中的图像一样，缓冲也可以由特定队列族拥有，或者在多个队列之间共享。由于缓冲仅将在图形队列中使用，因此我们可以使用独占访问。

```rust,noplaypen
    .flags(vk::BufferCreateFlags::empty()); // Optional.
```

`flags` 参数用于配置稀疏缓冲内存，这在当前情况下并不相关。您可以省略此字段的构建方法，从而将其设置为默认值（空标志集）。

现在，我们可以使用 `create_buffer` 创建缓冲。首先，定义一个 `AppData` 字段来保存缓冲句柄，并称其为 `vertex_buffer`。

```rust,noplaypen
struct AppData {
    // ...
    vertex_buffer: vk::Buffer,
}
```

接下来添加对 `create_buffer` 的调用：

```rust,noplaypen
unsafe fn create_vertex_buffer(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let buffer_info = vk::BufferCreateInfo::builder()
        .size((size_of::<Vertex>() * VERTICES.len()) as u64)
        .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
        .sharing_mode(vk::SharingMode::EXCLUSIVE);

    data.vertex_buffer = device.create_buffer(&buffer_info, None)?;

    Ok(())
}
```

缓冲应该在渲染命令中可用，直到程序结束，且它不依赖于交换链，因此我们将在原始的 `App::destroy` 方法中清理它：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_buffer(self.data.vertex_buffer, None);
    // ...
}
```

## 内存需求

缓冲已经创建了，但实际上还没有为其分配任何内存。为缓冲分配内存的第一步是使用名为 `get_buffer_memory_requirements` 的命令查询其内存需求。

```rust,noplaypen
let requirements = device.get_buffer_memory_requirements(data.vertex_buffer);
```

这个命令返回的 `vk::MemoryRequirements` 结构体有三个字段：

* `size` &ndash; 所需内存大小（以字节为单位），可能与 `bufferInfo.size` 不同。
* `alignment` &ndash; 缓冲在内存分配的区域中开始的偏移量（以字节为单位），取决于 `buffer_info.usage` 和 `buffer_info.flags`。
* `memory_type_bits` &ndash; 适用于缓冲的内存类型的位字段。

显卡可以提供不同类型的内存供分配。每种类型的内存在允许的操作和性能特性方面各不相同。我们需要将缓冲的需求和我们自己的应用程序需求结合起来，找到合适的内存类型。为此，我们创建一个新函数 `get_memory_type_index`。

```rust,noplaypen
unsafe fn get_memory_type_index(
    instance: &Instance,
    data: &AppData,
    properties: vk::MemoryPropertyFlags,
    requirements: vk::MemoryRequirements,
) -> Result<u32> {
}
```

首先，我们需要使用 `get_physical_device_memory_properties` 查询有关可用内存类型的信息。

```rust,noplaypen
let memory = instance.get_physical_device_memory_properties(data.physical_device);
```

返回的 `vk::PhysicalDeviceMemoryProperties` 结构体有两个数组 `memory_types` 和 `memory_heaps`。内存堆是不同的内存资源，比如专用的VRAM和用于VRAM耗尽时的RAM交换空间。不同类型的内存存在于这些堆中。现在我们只关注内存类型而不是它来自的堆，但可以想象这可能会影响性能。

首先，让我们找到一个适合缓冲本身的内存类型：

```rust,noplaypen
(0..memory.memory_type_count)
    .find(|i| (requirements.memory_type_bits & (1 << i)) != 0)
    .ok_or_else(|| anyhow!("Failed to find suitable memory type."))
```

`requirements` 参数中的 `memory_type_bits` 字段将用于指定适合的内存类型的位字段。这意味着我们可以通过简单地迭代并检查相应的位是否设置为 `1` 来找到适合的内存类型的索引。

然而，我们不仅对适合顶点缓冲的内存类型感兴趣，我们还需要能够将顶点数据写入该内存。`memory_types` 数组由指定每种内存类型的堆和属性的`vk::MemoryType` 结构体组成。属性定义了内存的特殊功能，例如是否可以从CPU映射它以便我们可以从CPU写入数据。这个属性通过 `vk::MemoryPropertyFlags::HOST_VISIBLE` 来指示，但我们还需要使用 `vk::MemoryPropertyFlags::HOST_COHERENT` 属性。我们将在映射内存时看到为什么需要这样做。

现在，我们可以修改循环以检查此属性的支持：

```rust,noplaypen
(0..memory.memory_type_count)
    .find(|i| {
        let suitable = (requirements.memory_type_bits & (1 << i)) != 0;
        let memory_type = memory.memory_types[*i as usize];
        suitable && memory_type.property_flags.contains(properties)
    })
    .ok_or_else(|| anyhow!("Failed to find suitable memory type."))
```

如果存在适合缓冲的内存类型，并且该内存类型具有我们所需的所有属性，则返回其索引；否则返回错误。

## 内存分配

现在我们有一种确定正确内存类型的方法，因此我们可以通过填充 `vk::MemoryAllocateInfo` 结构体来实际分配内存。

```rust,noplaypen
let memory_info = vk::MemoryAllocateInfo::builder()
    .allocation_size(requirements.size)
    .memory_type_index(get_memory_type_index(
        instance,
        data,
        vk::MemoryPropertyFlags::HOST_COHERENT | vk::MemoryPropertyFlags::HOST_VISIBLE,
        requirements,
    )?);
```

内存分配现在就是简单地指定大小和类型，这两者都来自于顶点缓冲的内存需求和所需的属性。创建一个 `AppData` 字段来存储内存的句柄：

```rust,noplaypen
struct AppData {
    // ...
    vertex_buffer: vk::Buffer,
    vertex_buffer_memory: vk::DeviceMemory,
}
```

通过调用 `allocate_memory` 来填充这个新字段：

```rust,noplaypen
data.vertex_buffer_memory = device.allocate_memory(&memory_info, None)?;
```

如果内存分配成功，那么我们现在可以使用 `bind_buffer_memory` 将此内存与缓冲关联：

```rust,noplaypen
device.bind_buffer_memory(data.vertex_buffer, data.vertex_buffer_memory, 0)?;
```

前两个参数不言自明，第三个参数是在内存区域内的偏移量。由于此内存专门为该顶点缓冲分配，因此偏移量只是 `0` 。如果偏移量为非零值，则必须可被 `requirements.alignment` 整除。

当然，就像在C中动态内存分配一样，内存应该在某个时候被释放。绑定到缓冲对象的内存在缓冲不再使用时可以被释放，所以让我们在缓冲被销毁后释放它：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_buffer(self.data.vertex_buffer, None);
    self.device.free_memory(self.data.vertex_buffer_memory, None);
    // ...
}
```

## 填充顶点缓冲

现在是时候将顶点数据复制到缓冲了。这是通过将缓冲内存映射到CPU可访问的内存中来完成的，可以使用 `map_memory` 命令。

```rust,noplaypen
let memory = device.map_memory(
    data.vertex_buffer_memory,
    0,
    buffer_info.size,
    vk::MemoryMapFlags::empty(),
)?;
```

该命令允许我们访问由偏移量和大小定义的指定内存资源的区域。在这里，偏移量和大小分别为 `0` 和 `buffer_info.size`。还可以使用特殊值 `vk::WHOLE_SIZE` 来映射所有内存。最后一个参数可用于指定标志，但当前API中还没有任何可用的标志。它必须设置为空标志集。返回的值是映射值的指针。

在继续之前，我们需要能够将顶点列表的内存复制到映射内存中。在程序中添加这个导入：

```rust,noplaypen
use std::ptr::copy_nonoverlapping as memcpy;
```

现在我们可以将顶点数据复制到缓冲内存中，然后使用 `unmap_memory` 将其取消映射。

```rust,noplaypen
memcpy(VERTICES.as_ptr(), memory.cast(), VERTICES.len());
device.unmap_memory(data.vertex_buffer_memory);
```

不幸的是，驱动程序可能不会立即将数据复制到缓冲内存中，例如因为缓存的原因。还有可能写入缓冲的数据尚未在映射内存中可见。有两种方法可以解决这个问题：

* 使用内存堆是主机一致的，使用 `vk::MemoryPropertyFlags::HOST_COHERENT` 表示
* 在写入映射内存后调用 `flush_mapped_memory_ranges`，在从映射内存读取之前调用 `invalidate_mapped_memory_ranges`

我们采用了第一种方法，这样可以确保映射内存始终与分配的内存内容相匹配。请记住，这可能会导致稍微较差的性能，但我们将在下一章看到为什么这没关系。

刷新内存范围或使用一致性内存堆意味着驱动程序将知道我们对缓冲的写入，但并不意味着它们实际上已经在GPU上可见。数据传输到GPU是在后台进行的操作，规范仅告诉我们，它在下一次 `queue_submit` 调用时是保证完成的。

## 绑定顶点缓冲

现在，唯一剩下的任务是在渲染操作期间绑定顶点缓冲。我们将扩展 `create_command_buffers` 函数来完成这个任务。

```rust,noplaypen
// ...
device.cmd_bind_vertex_buffers(*command_buffer, 0, &[data.vertex_buffer], &[0]);
device.cmd_draw(*command_buffer, VERTICES.len() as u32, 1, 0, 0);
// ...
```

`cmd_bind_vertex_buffers` 命令用于将顶点缓冲绑定到绑定点，就像我们在上一章中设置的那样。第二个参数指定我们正在使用的顶点输入绑定的索引。最后两个参数指定要绑定的顶点缓冲和从中开始读取顶点数据的字节偏移量。您还应该更改对 `cmd_draw` 的调用，将缓冲中的顶点数传递给该函数，而不是硬编码的数字 `3`。

现在运行程序，您应该再次看到熟悉的三角形：

![三角形](../images/triangle.png)

通过修改 `VERTICES` 列表，将顶点的颜色更改为白色，可以尝试修改三角形的顶点颜色：

```rust,noplaypen
lazy_static! {
    static ref VERTICES: Vec<Vertex> = vec![
        Vertex::new(glm::vec2(0.0, -0.5), glm::vec3(1.0, 1.0, 1.0)),
        Vertex::new(glm::vec2(0.5, 0.5), glm::vec3(0.0, 1.0, 0.0)),
        Vertex::new(glm::vec2(-0.5, 0.5), glm::vec3(0.0, 0.0, 1.0)),
    ];
}
```

再次运行程序，您应该会看到以下效果：

![白色三角](../images/triangle_white.png)

在下一章中，我们将介绍一种不同的将顶点数据复制到顶点缓冲的方法，这将导致更好的性能，但需要更多的工作。
