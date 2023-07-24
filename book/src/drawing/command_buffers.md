# 指令缓冲（Command buffers）

> 原文链接：<https://kylemayes.github.io/vulkanalia/drawing/command_buffers.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码：** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/14_command_buffers.rs)

Vulkan 中的指令 —— 例如绘制操作和内存传输操作 —— 并不是直接通过调用函数来执行的。你需要把你想执行的操作记录在指令缓冲对象中。这样做的优势在于绘制指令可以提前配置好，并且可以在多个线程中配置指令。在配置完指令缓冲之后，你只要在主循环中高速 Vulkan 执行这些指令就可以了。

## 指令池（Command pools）

在创建指令缓冲之前，我们需要先创建一个指令池。指令池管理着用于存储指令缓冲的内存，我们将从指令池中分配指令缓冲。在 `AppData` 中添加一个新的字段 `vk::CommandPool` 来存储指令池：

We have to create a command pool before we can create command buffers. Command pools manage the memory that is used to store the buffers and command buffers are allocated from them. Add a new `AppData` field to store a `vk::CommandPool`:

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

指令缓冲是通过提交到一个设备队列来执行的。每个指令池只能提交到单一类型的队列。这里我们要记录绘制指令，所以我们选择图形队列族。

指令池可以有三个标志：

* `vk::CommandPoolCreateFlags::TRANSIENT` &ndash; 指令缓冲会经常被重新记录（可能会改变内存分配行为）
* `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` &ndash; 允许单独重新记录指令缓冲，如果没有这个标志，所有指令缓冲都必须一起重置
* `vk::CommandPoolCreateFlags::PROTECTED` &ndash; 创建“受保护”的指令缓冲，它们存储在[“受保护”内存](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#memory-protected-access-rules)中，Vulkan 会阻止对该内存未授权的访问

我们只在程序开始的时候记录指令缓冲，然后在主循环中重复执行它们，并且我们也不需要使用 DRM 来保护我们的三角形，所以我们不使用任何标志。

```rust,noplaypen
data.command_pool = device.create_command_pool(&info, None)?;
```

从指令池分配的指令缓冲会在整个程序中被使用，所以我们在 `destroy` 函数中销毁它：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_command_pool(self.data.command_pool, None);
    // ...
}
```

## 分配指令缓冲

We can now start allocating command buffers and recording drawing commands in them. Because one of the drawing commands involves binding the right `vk::Framebuffer`, we'll actually have to record a command buffer for every image in the swapchain once again. To that end, create a list of `vk::CommandBuffer` objects as an `AppData` field. Command buffers will be automatically freed when their command pool is destroyed, so we don't need any explicit cleanup.

```rust,noplaypen
struct AppData {
    // ...
    command_buffers: Vec<vk::CommandBuffer>,
}
```

We'll now start working on a `create_command_buffers` function that allocates and records the commands for each swapchain image.

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

Command buffers are allocated with the `allocate_command_buffers` function, which takes a `vk::CommandBufferAllocateInfo` struct as parameter that specifies the command pool and number of buffers to allocate:

```rust,noplaypen
let allocate_info = vk::CommandBufferAllocateInfo::builder()
    .command_pool(data.command_pool)
    .level(vk::CommandBufferLevel::PRIMARY)
    .command_buffer_count(data.framebuffers.len() as u32);

data.command_buffers = device.allocate_command_buffers(&allocate_info)?;
```

The `level` parameter specifies if the allocated command buffers are primary or secondary command buffers.

* `vk::CommandBufferLevel::PRIMARY` &ndash; Can be submitted to a queue for execution, but cannot be called from other command buffers.
* `vk::CommandBufferLevel::SECONDARY` &ndash; Cannot be submitted directly, but can be called from primary command buffers.

We won't make use of the secondary command buffer functionality here, but you can imagine that it's helpful to reuse common operations from primary command buffers.

## Starting command buffer recording

We begin recording a command buffer by calling `begin_command_buffer` with a small `vk::CommandBufferBeginInfo` structure as argument that specifies some details about the usage of this specific command buffer.

```rust,noplaypen
for (i, command_buffer) in data.command_buffers.iter().enumerate() {
    let inheritance = vk::CommandBufferInheritanceInfo::builder();

    let info = vk::CommandBufferBeginInfo::builder()
        .flags(vk::CommandBufferUsageFlags::empty()) // Optional.
        .inheritance_info(&inheritance);             // Optional.

    device.begin_command_buffer(*command_buffer, &info)?;
}
```

The `flags` parameter specifies how we're going to use the command buffer. The following values are available:

* `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` &ndash; The command buffer will be rerecorded right after executing it once.
* `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE` &ndash; This is a secondary command buffer that will be entirely within a single render pass.
* `vk::CommandBufferUsageFlags::SIMULTANEOUS_USE` &ndash; The command buffer can be resubmitted while it is also already pending execution.

None of these flags are applicable for us right now.

The `inheritance_info` parameter is only relevant for secondary command buffers. It specifies which state to inherit from the calling primary command buffers.

If the command buffer was already recorded once, then a call to `begin_command_buffer` will implicitly reset it. It's not possible to append commands to a buffer at a later time.

## Starting a render pass

Before we can start a render pass we'll need to build some parameters.

```rust,noplaypen
let render_area = vk::Rect2D::builder()
    .offset(vk::Offset2D::default())
    .extent(data.swapchain_extent);
```

Here we define the size of the render area. The render area defines where shader loads and stores will take place during the execution of the render pass. The pixels outside this region will have undefined values. It should match the size of the attachments for best performance.

```rust,noplaypen
let color_clear_value = vk::ClearValue {
    color: vk::ClearColorValue {
        float32: [0.0, 0.0, 0.0, 1.0],
    },
};
```

Next we define a clear value that will be used to clear the framebuffer at the beginning of the render pass (because we used `vk::AttachmentLoadOp::CLEAR` when creating the render pass). `vk::ClearValue` is a union that can be used to set clear values for color attachments or for depth/stencil attachments. Here we are setting the `color` field with a `vk::ClearColorValue` union with 4 `f32`s that define a black clear color with 100% opacity.

Drawing starts by beginning the render pass with `cmd_begin_render_pass`. The render pass is configured using some parameters in a `vk::RenderPassBeginInfo` struct.

```rust,noplaypen
let clear_values = &[color_clear_value];
let info = vk::RenderPassBeginInfo::builder()
    .render_pass(data.render_pass)
    .framebuffer(data.framebuffers[i])
    .render_area(render_area)
    .clear_values(clear_values);
```

The first parameters are the render pass itself and the attachments to bind. We created a framebuffer for each swapchain image that specifies it as color attachment. Then we provide the previously constructed render area and clear value.

```rust,noplaypen
device.cmd_begin_render_pass(
    *command_buffer, &info, vk::SubpassContents::INLINE);
```

The render pass can now begin. All of the functions that record commands can be recognized by their `cmd_` prefix. They all return `()`, so there is no need for error handling until we've finished recording.

The first parameter for every command is always the command buffer to record the command to. The second parameter specifies the details of the render pass we've just provided. The final parameter controls how the drawing commands within the render pass will be provided. It can have one of two values:

* `vk::SubpassContents::INLINE` &ndash; The render pass commands will be embedded in the primary command buffer itself and no secondary command buffers will be executed.
* `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS` &ndash; The render pass commands will be executed from secondary command buffers.

We will not be using secondary command buffers, so we'll go with the first option.

## Basic drawing commands

We can now bind the graphics pipeline:

```rust,noplaypen
device.cmd_bind_pipeline(
    *command_buffer, vk::PipelineBindPoint::GRAPHICS, data.pipeline);
```

The second parameter specifies if the pipeline object is a graphics or compute pipeline. We've now told Vulkan which operations to execute in the graphics pipeline and which attachment to use in the fragment shader, so all that remains is telling it to draw the triangle:

```rust,noplaypen
device.cmd_draw(*command_buffer, 3, 1, 0, 0);
```

The actual drawing function is a bit anticlimactic, but it's so simple because of all the information we specified in advance. It has the following parameters, aside from the command buffer:

* `vertex_count` &ndash; Even though we don't have a vertex buffer, we technically still have 3 vertices to draw.
* `instance_count` &ndash; Used for instanced rendering, use `1` if you're not doing that.
* `first_vertex` &ndash; Used as an offset into the vertex buffer, defines the lowest value of `gl_VertexIndex`.
* `first_instance` &ndash; Used as an offset for instanced rendering, defines the lowest value of `gl_InstanceIndex`.

## Finishing up

The render pass can now be ended:

```rust,noplaypen
device.cmd_end_render_pass(*command_buffer);
```

And we've finished recording the command buffer:

```rust,noplaypen
device.end_command_buffer(*command_buffer)?;
```

In the next chapter we'll write the code for the main loop, which will acquire an image from the swapchain, execute the right command buffer and return the finished image to the swapchain.
