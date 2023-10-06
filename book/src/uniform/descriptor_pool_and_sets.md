# 描述符池与描述符集合

> 原文链接：<https://kylemayes.github.io/vulkanalia/uniform/descriptor_pool_and_sets.html>
>
> Commit Hash: ceb4a3fc6d8ca565af4f8679c4889bcad7941338

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/22_descriptor_sets.rs)

上一章提到的描述符集合布局描述了可以绑定的描述符的类型。在本章中，我们将为每个 `vk::Buffer` 资源创建一个描述符集合，以将其绑定到 uniform 缓冲描述符。

## 描述符池

描述符集合不能被直接创建，而是像指令缓冲一样必须从池中分配。类比指令池之于指令缓冲，描述符集合的等效物不出所料地被称为*描述符池*。我们将编写一个新的函数 `create_descriptor_pool` 来设置它。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_uniform_buffers(&instance, &device, &mut data)?;
        create_descriptor_pool(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_descriptor_pool(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

首先我们需要使用 `vk::DescriptorPoolSize` 结构描述我们的描述符集合将包含哪些描述符类型以及它们的数量。

```rust,noplaypen
let ubo_size = vk::DescriptorPoolSize::builder()
    .type_(vk::DescriptorType::UNIFORM_BUFFER)
    .descriptor_count(data.swapchain_images.len() as u32);
```

我们会为每一帧分配一个这样的描述符。包含了最大描述符集合数量信息的 `vk::DescriptorPoolSize` 结构会被主要的 `vk::DescriptorPoolCreateInfo` 引用：

```rust,noplaypen
let pool_sizes = &[ubo_size];
let info = vk::DescriptorPoolCreateInfo::builder()
    .pool_sizes(pool_sizes)
    .max_sets(data.swapchain_images.len() as u32);
```

类似于指令池，这个结构有一个的可选标志 `vk::DescriptorPoolCreateFlags::FREE_DESCRIPTOR_SET`，用于确定是否可以释放单个描述符集合。我们在创建描述符集合后不会再修改它，所以我们不需要这个标志。

```rust,noplaypen
struct AppData {
    // ...
    uniform_buffers: Vec<vk::Buffer>,
    uniform_buffers_memory: Vec<vk::DeviceMemory>,
    descriptor_pool: vk::DescriptorPool,
    // ...
}
```

在 `AppData` 中添加一个新的字段来存储描述符池的句柄，并调用 `create_descriptor_pool` 来创建它。

```rust,noplaypen
data.descriptor_pool = device.create_descriptor_pool(&info, None)?;
```

当重建交换链时，应该销毁描述符池，因为它取决于图像的数量：

```rust,noplaypen
unsafe fn destroy_swapchain(&mut self) {
    self.device.destroy_descriptor_pool(self.data.descriptor_pool, None);
    // ...
}
```

并且在 `App::recreate_swapchain` 中重新创建描述符池：

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    // ...
    create_uniform_buffers(&self.instance, &self.device, &mut self.data)?;
    create_descriptor_pool(&self.device, &mut self.data)?;
    // ...
}
```

## 描述符集合

现在我们可以分配描述符集合本身了。添加一个 `create_descriptor_sets` 函数：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_descriptor_pool(&device, &mut data)?;
        create_descriptor_sets(&device, &mut data)?;
        // ...
    }

    unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
        // ..
        create_descriptor_pool(&self.device, &mut self.data)?;
        create_descriptor_sets(&self.device, &mut self.data)?;
        // ..
    }
}

unsafe fn create_descriptor_sets(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

描述符集合分配使用 `vk::DescriptorSetAllocateInfo` 结构描述。你需要指定要分配的描述符池，以及描述符集合布局的数组，该数组描述了要分配的每个描述符集合：

```rust,noplaypen
let layouts = vec![data.descriptor_set_layout; data.swapchain_images.len()];
let info = vk::DescriptorSetAllocateInfo::builder()
    .descriptor_pool(data.descriptor_pool)
    .set_layouts(&layouts);
```

在我们的例子中，我们将为每个交换链图像创建一个描述符集合，所有的描述符集合都具有相同的布局。不幸的是，我们只能把描述符集合布局复制多次，因为 `set_layouts` 字段需要一个与描述符集合数量相匹配的数组。

在 `AppData` 中添加一个字段来保存描述符集合的句柄：

```rust,noplaypen
struct AppData {
    // ...
    descriptor_pool: vk::DescriptorPool,
    descriptor_sets: Vec<vk::DescriptorSet>,
    // ...
}
```

并使用 `allocate_descriptor_sets` 分配它们：

```rust,noplaypen
data.descriptor_sets = device.allocate_descriptor_sets(&info)?;
```

你不需要显式地清理描述符集合，因为当描述符池被销毁时，它们将自动释放。调用 `allocate_descriptor_sets` 将分配描述符集合，每个集合都有一个 uniform 缓冲描述符。

现在已经分配了描述符集合，但是其中的描述符仍然需要配置。我们现在将添加一个循环来填充每个描述符：

```rust,noplaypen
for i in 0..data.swapchain_images.len() {

}
```

指向缓冲的描述符 —— 例如我们的 uniform 缓冲描述符 —— 使用 `vk::DescriptorBufferInfo` 结构进行配置。该结构指定了缓冲以及其中包含描述符数据的区域。

```rust,noplaypen
for i in 0..data.swapchain_images.len() {
    let info = vk::DescriptorBufferInfo::builder()
        .buffer(data.uniform_buffers[i])
        .offset(0)
        .range(size_of::<UniformBufferObject>() as u64);
}
```

如果你要覆盖整个缓冲，就像我们在这个例子中一样，你也可以使用 `vk::WHOLE_SIZE` 值来表示范围。描述符的配置使用 `update_descriptor_sets` 函数进行更新，该函数以 `vk::WriteDescriptorSet` 结构的数组作为参数。

```rust,noplaypen
let buffer_info = &[info];
let ubo_write = vk::WriteDescriptorSet::builder()
    .dst_set(data.descriptor_sets[i])
    .dst_binding(0)
    .dst_array_element(0)
    // continued...
```

前两个字段指定了要更新的描述符集合和绑定。我们给 uniform 缓冲绑定索引 `0`。请记住，描述符可以是数组，因此我们还需要用 `dst_array_element` 字段指定要更新的数组中的第一个索引。我们没有使用数组，所以索引是 `0`。

```rust,noplaypen
    .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
```

我们需要再次指定描述符的类型。

```rust,noplaypen
    .buffer_info(buffer_info);
```

最后一个字段引用了一个数组，其中包含 `descriptor_count` 个实际配置描述符的结构体。取决于描述符的类型，你需要使用以下三个字段之一：`buffer_info` 字段用于指向缓冲数据的描述符，`image_info` 用于指向图像数据的描述符，`texel_buffer_view` 用于指向缓冲视图的描述符。我们的描述符基于缓冲，所以我们使用 `buffer_info`。

```rust,noplaypen
device.update_descriptor_sets(&[ubo_write], &[] as &[vk::CopyDescriptorSet]);
```

使用 `update_descriptor_sets` 应用更新。它接受两种类型的数组作为参数：`vk::WriteDescriptorSet` 数组和 `vk::CopyDescriptorSet` 数组。后者可以用来将描述符复制到彼此，正如它的名字所暗示的那样。

## 使用描述符集合

现在我们需要更新 `create_command_buffers` 函数，使用 `cmd_bind_descriptor_sets` 来将每个与交换链图像对应的描述符集合绑定到着色器中的描述符上。这需要在 `cmd_draw_indexed` 调用之前完成：

```rust,noplaypen
device.cmd_bind_descriptor_sets(
    *command_buffer,
    vk::PipelineBindPoint::GRAPHICS,
    data.pipeline_layout,
    0,
    &[data.descriptor_sets[i]],
    &[],
);
device.cmd_draw_indexed(*command_buffer, INDICES.len() as u32, 1, 0, 0, 0);
```

不同于顶点和索引缓冲的是，描述符集合并不是专为图形管线而设的。因此我们需要指定我们想要将描述符集合绑定到图形管线还是计算管线。下一个参数是描述符基于的管线布局。接下来的两个参数指定了第一个描述符集合的索引和要绑定的集合数组。我们稍后会回到这个问题。最后一个参数指定了用于动态描述符的偏移量数组。我们将在后面的章节中看到这些。

如果你现在运行程序，那么很不幸，你看不到任何东西。问题在于我们在投影矩阵中翻转了 Y 坐标，现在顶点是按逆时针顺序而不是顺时针顺序绘制的。这导致背面剔除生效，并阻止任何几何图形被绘制。进入 `create_pipeline` 函数并修改 `vk::PipelineRasterizationStateCreateInfo` 中的 `front_face` 以纠正这个问题：

```rust,noplaypen
    .cull_mode(vk::CullModeFlags::BACK)
    .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
```

再次运行程序，你应该能看到以下内容：

![](../images/spinning_quad.png)

矩形变成了方形，因为投影矩阵现在纠正了宽高比。`App::update_uniform_buffer` 方法负责屏幕调整大小，所以我们不需要在 `App::recreate_swapchain` 中重新创建描述符集合。

## 对齐要求

到目前为止，我们忽略了一个问题，那就是 Rust 结构中的数据应该如何与着色器中的 uniform 定义匹配。在两者中使用相同的类型似乎是显而易见的：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    model: Mat4,
    view: Mat4,
    proj: Mat4,
}
```

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

然而这不只是全部。例如，试试看像这样修改结构和着色器：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    foo: Vec2,
    model: Mat4,
    view: Mat4,
    proj: Mat4,
}
```

```glsl
layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

重新编译你的着色器和程序并运行它，你会发现五颜六色的正方形消失了！这是因为我们没有考虑到*对齐要求（alignment requirements）*。

Vulkan 希望你的结构中的数据在内存中以特定的方式对齐，例如：

* 标量必须以 N (= 4 字节，给定 32 位浮点数) 对齐。
* `vec2` 必须以 2N (= 8 字节) 对齐。
* `vec3` 或 `vec4` 必须以 4N (= 16 字节) 对齐。
* 嵌套结构必须以其成员的基本对齐方式对齐，向上舍入为 16 的倍数。
* `mat4` 矩阵必须与 `vec4` 具有相同的对齐方式。

你可以在[规范](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/chap14.html#interfaces-resources-layout)中找到完整的对齐要求列表。

我们原先的着色器只有三个 `mat4` 字段，已经满足了对齐要求。由于每个 `mat4` 的大小为 4 x 4 x 4 = 64 字节，`model` 的偏移量为 `0`，`view` 的偏移量为 64，`proj` 的偏移量为 128。所有这些都是 16 的倍数，这就是为什么它碰巧能正常工作。

而新的结构则以 `vec2` 开头，而 `vec2` 只有 8 字节大小，因此后面所有的偏移量都会被打乱。现在 `model` 的偏移量为 `8`，`view` 的偏移量为 `72`，`proj` 的偏移量为 `136`，它们都不是 16 的倍数。不幸的是，Rust 对于控制结构体字段的对齐方式没有很好的支持，但是我们可以手动填充来修复对齐问题：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    foo: Vec2,
    _padding: [u8; 8],
    model: Mat4,
    view: Mat4,
    proj: Mat4,
}
```

如果你现在重新编译并再次运行程序，你应该会看到着色器再次正确地接收到矩阵值。

## 多个描述符集合

正如一些结构和函数调用所暗示的那样，实际上可以同时绑定多个描述符集合。在创建管线布局时，你需要为每个描述符集合指定一个描述符集合布局。然后着色器可以像这样引用特定的描述符集合：

```glsl
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

你可以利用这个特性，将每个对象都不同的描述符和在对象之间共享的描述符分别放入不同的描述符集合中。在这种情况下，你可以避免在绘制调用之间重新绑定大多数描述符，这可能更有效率。
