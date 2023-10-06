# 指令缓冲（Command buffers）

> 原文链接：<https://kylemayes.github.io/vulkanalia/drawing/command_buffers.html>
>
> Commit Hash: ceb4a3fc6d8ca565af4f8679c4889bcad7941338

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/14_command_buffers.rs)

Vulkan 中的指令 —— 例如绘制操作和内存传输操作 —— 并不是通过直接调用函数来执行的。你需要把你想执行的操作记录在指令缓冲对象中。这样做的优势在于绘制指令可以提前配置好，并且可以在多个线程中配置指令。在配置完指令缓冲之后，你只要在主循环中告诉 Vulkan 执行这些指令就可以了。

## 指令池（Command pools）

在创建指令缓冲之前，我们需要先创建一个指令池。指令池管理着用于存储指令缓冲的内存，我们将从指令池中分配指令缓冲。在 `AppData` 中添加一个新的字段 `vk::CommandPool` 来存储指令池：

```rust,noplaypen
struct AppData {
    // ...
    command_pool: vk::CommandPool,
}
```

接着创建一个新的函数 `create_command_pool` 并在 `App::create` 中创建完帧缓冲后调用它：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_framebuffers(&device, &mut data)?;
        create_command_pool(&instance, &device, &mut data)?;
        // ...
    }
}

unsafe fn create_command_pool(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

创建指令池只需要两个参数：

```rust,noplaypen
let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;

let info = vk::CommandPoolCreateInfo::builder()
    .flags(vk::CommandPoolCreateFlags::empty()) // 可选
    .queue_family_index(indices.graphics);
```

指令缓冲是通过提交到一个设备队列 —— 例如图形队列或呈现队列 —— 来执行的。每个指令池分配的指令缓冲只能提交到一种队列。这里我们要记录用于绘制的指令，所以我们选择图形队列族。

指令池可以有三个标志：

* `vk::CommandPoolCreateFlags::TRANSIENT` &ndash; 提示指令缓冲会经常被重新记录（可能会改变内存分配行为）
* `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` &ndash; 允许单独重新记录指令缓冲，如果没有这个标志，所有指令缓冲都必须一起重置
* `vk::CommandPoolCreateFlags::PROTECTED` &ndash; 创建“受保护”的指令缓冲，它们存储在[“受保护”内存](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#memory-protected-access-rules)中，Vulkan 会阻止对该内存未授权的访问

我们只在程序开始的时候记录指令缓冲，然后在主循环中重复执行它们，并且我们也不需要使用 DRM 来保护我们的三角形，所以我们不使用任何标志。

```rust,noplaypen
data.command_pool = device.create_command_pool(&info, None)?;
```

从指令池分配的指令缓冲会在整个程序中被使用，所以缓冲池应该在程序结束时销毁：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_command_pool(self.data.command_pool, None);
    // ...
}
```

## 分配指令缓冲

现在我们可以开始分配指令缓冲，并在其中记录绘制指令了。因为某个绘制指令涉及到绑定正确的 `vk::Framebuffer`，所以我们实际上要为交换链中的每张图像都记录一个指令缓冲。为此，我们在 `AppData` 中创建一个 `vk::CommandBuffer` 对象的列表。指令缓冲会在它们所属的指令池被销毁时自动释放，所以我们不需要进行显式的清理。

```rust,noplaypen
struct AppData {
    // ...
    command_buffers: Vec<vk::CommandBuffer>,
}
```

接下来我们开始实现用于分配并记录指令缓冲的 `create_command_buffers` 函数。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_command_pool(&instance, &device, &mut data)?;
        create_command_buffers(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_command_buffers(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

指令缓冲由 `allocate_command_buffers` 函数分配，它接受一个 `vk::CommandBufferAllocateInfo` 结构体作为参数，这个结构体指定了指令池和要分配的指令缓冲的数量：

```rust,noplaypen
let allocate_info = vk::CommandBufferAllocateInfo::builder()
    .command_pool(data.command_pool)
    .level(vk::CommandBufferLevel::PRIMARY)
    .command_buffer_count(data.framebuffers.len() as u32);

data.command_buffers = device.allocate_command_buffers(&allocate_info)?;
```

`level` 参数指定了分配的指令缓冲是主指令缓冲还是次级指令缓冲。

* `vk::CommandBufferLevel::PRIMARY` &ndash; 可以提交到队列执行，但不能从其他指令缓冲中调用
* `vk::CommandBufferLevel::SECONDARY` &ndash; 不能直接提交，但可以从主指令缓冲中调用

这里我们用不到次级指令缓冲，不过不过你能想到，次级指令缓冲对于复用主指令缓冲中的常用操作很有帮助。

## 开始记录指令缓冲

我们调用 `begin_command_buffer` 函数来开始记录指令缓冲，它接受一个 `vk::CommandBufferBeginInfo` 结构体作为参数，这个结构体指定一些有关指令缓冲使用方式的细节。

```rust,noplaypen
for (i, command_buffer) in data.command_buffers.iter().enumerate() {
    let inheritance = vk::CommandBufferInheritanceInfo::builder();

    let info = vk::CommandBufferBeginInfo::builder()
        .flags(vk::CommandBufferUsageFlags::empty()) // 可选
        .inheritance_info(&inheritance);             // 可选

    device.begin_command_buffer(*command_buffer, &info)?;
}
```

`flag` 参数指定了我们将要如何使用这个指令缓冲，它可以有以下取值：

* `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` &ndash; 指令缓冲会在执行一次之后重新记录
* `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE` &ndash; 这是一个次级指令缓冲，它会完全在一个渲染流程中执行
* `vk::CommandBufferUsageFlags::SIMULTANEOUS_USE` &ndash; 指令缓冲可以在它还在等待执行的时候被重新提交

目前我们还不需要这些标志。

`inheritance_info` 参数只用于次级指令缓冲，它指定了要从调用它的主指令缓冲中继承哪些状态。

如果指令缓冲已经被记录过一次，调用 `begin_command_buffer` 会隐式地重置它。一旦记录完成，就不能再向指令缓冲中追加指令了。

## 开始渲染流程

在我们开始渲染流程之前，我们需要先构建一些参数。

```rust,noplaypen
let render_area = vk::Rect2D::builder()
    .offset(vk::Offset2D::default())
    .extent(data.swapchain_extent);
```

这里我们定义了渲染区域的大小。渲染区域定义了在渲染流程执行期间着色器会在哪里加载和存储像素。渲染区域之外的像素的值是未定义的。渲染区域应该和附件的大小匹配以获得最佳性能。

```rust,noplaypen
let color_clear_value = vk::ClearValue {
    color: vk::ClearColorValue {
        float32: [0.0, 0.0, 0.0, 1.0],
    },
};
```

接着我们定义一个清除值，它会被用来在渲染流程开始时清空帧缓冲（因为我们在创建渲染流程的时候指定了 `vk::AttachmentLoadOp::CLEAR`）。`vk::ClearValue` 是一个联合体（union），它可以用来设置颜色附件的清除值，也可以用来设置深度/模板附件的清除值。这里我们设置了 `vk::ClearColorValue` 类型的 `color` 字段，用来将清除颜色设为不透明的黑色。

绘制以 `cmd_begin_render_pass` 启动渲染流程开始，渲染流程由 `vk::RenderPassBeginInfo` 结构体来配置：

```rust,noplaypen
let clear_values = &[color_clear_value];
let info = vk::RenderPassBeginInfo::builder()
    .render_pass(data.render_pass)
    .framebuffer(data.framebuffers[i])
    .render_area(render_area)
    .clear_values(clear_values);
```

首先我们提供渲染流程和将要绑定的附件。之前，我们为交换链中的每个图像都创建了一个帧缓冲，用作颜色附件。然后我们提供刚才创建的渲染区域和清除值。

```rust,noplaypen
device.cmd_begin_render_pass(
    *command_buffer, &info, vk::SubpassContents::INLINE);
```

现在渲染流程可以开始了。所有记录指令的函数都以 `cmd_` 前缀开头。它们都返回 `()`，所以所以我们在完成记录之前都不需要进行错误处理。

每个记录指令的函数的第一个参数都是用来记录指令的指令缓冲。第二个参数指定刚才提供的的渲染流程的细节。最后一个参数控制渲染流程中的绘制指令是如何提供的。它可以有以下两个值：

* `vk::SubpassContents::INLINE` &ndash; 渲染流程中的指令会被嵌入到主指令缓冲中，不会执行任何次级指令缓冲
* `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS` &ndash; 渲染流程中的指令会被从次级指令缓冲中执行

我们不会使用次级指令缓冲，所以我们选择第一个选项。

## 基本绘制指令

现在我们可以绑定图形管线：

```rust,noplaypen
device.cmd_bind_pipeline(
    *command_buffer, vk::PipelineBindPoint::GRAPHICS, data.pipeline);
```

第二个参数指定了管线对象是图形管线还是计算管线。至此，我们已经告诉 Vulkan 在图形管线中执行哪些操作，以及在片元着色器中使用哪个附件，剩下的就是告诉它绘制三角形：

```rust,noplaypen
device.cmd_draw(*command_buffer, 3, 1, 0, 0);
```

这个实际的绘制函数有点虎头蛇尾。我们之前提供了那么多信息，实际的绘制函数却如此简单。除了指令缓冲之外，它还有以下参数：

* `vertex_count` &ndash; 尽管我们没有顶点缓冲，技术上来说，我们是要绘制 3 个顶点。
* `instance_count` &ndash; 用于实例化渲染，如果你没在进行实例化渲染，就把它设为 `1`。
* `first_vertex` &ndash; 顶点缓冲的偏移量，定义了 `gl_VertexIndex` 的最小值。
* `first_instance` &ndash; 实例化渲染的偏移量，定义了 `gl_InstanceIndex` 的最小值。

## 完成

最后，我们调用 `cmd_end_render_pass` 函数结束渲染流程：

```rust,noplaypen
device.cmd_end_render_pass(*command_buffer);
```

并调用 `end_command_buffer` 结束记录指令缓冲：

```rust,noplaypen
device.end_command_buffer(*command_buffer)?;
```

在下一章中，我们将编写主循环的代码，它将从交换链中获取图像，执行正确的指令缓冲，并将完成的图像返回给交换链。
