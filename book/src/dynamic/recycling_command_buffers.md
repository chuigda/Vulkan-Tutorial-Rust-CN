# 重用指令缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/dynamic/recycling_command_buffers.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>The previous chapters of this tutorial that are not marked by this disclaimer were directly adapted from <https://github.com/Overv/VulkanTutorial>.<br/><br/>This chapter and the following chapters are instead original creations from someone who is most decidedly not an expert in Vulkan. An authoritative tone has been maintained, but these chapters should be considered a "best effort" by someone still learning Vulkan.<br/><br/>If you have questions, suggestions, or corrections, please [open an issue](https://github.com/KyleMayes/vulkanalia/issues)!

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>前面没有这个声明的章节都是直接从 <https://github.com/Overv/VulkanTutorial> 改编而来。<br/><br/>这一章和后面的章节都是原创，作者并不是 Vulkan 的专家。作者尽力保持了权威的语气，但是这些章节应该被视为一个 Vulkan 初学者的“尽力而为”。<br/><br/>如果你有问题、建议或者修正，请[提交 issue](https://github.com/KyleMayes/vulkanalia/issues)！

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/31_recycling_command_buffers.rs)

When you allocate a command buffer and record commands to it, Vulkan allocates blocks of memory to store information about the command buffer and the commands that have been recorded to it. Now that we want to be able to record different commands each frame, we need to recycle this memory in the same way that in C we need to `free` memory allocated with `malloc` once it is no longer in use.

当你分配指令缓冲并在其中记录指令的时候，Vulkan 会分配一块内存来存储指令缓冲的信息和已经记录到其中的指令。现在我们想要每一帧都记录不同的指令，我们需要回收这块内存，就像在 C 语言中我们需要在不再使用 `malloc` 分配的内存时使用 `free` 一样。

## 解决方案

Vulkan offers [three basic approaches](https://github.com/KhronosGroup/Vulkan-Samples/blob/524cdcd27005e7cd56e6694fa41e685519d7dbca/samples/performance/command_buffer_usage/command_buffer_usage_tutorial.md#recycling-strategies) for recycling the memory occupied by a command buffer:

Vulkan 为重用指令缓冲的内存提供了[三种基本的方式](https://github.com/KhronosGroup/Vulkan-Samples/blob/524cdcd27005e7cd56e6694fa41e685519d7dbca/samples/performance/command_buffer_usage/command_buffer_usage_tutorial.md#recycling-strategies)

1. Reset the command buffer (which clears the commands recorded to it) and record new commands to the command buffer
2. Free the command buffer (which returns its memory to the command pool it was allocated from) and allocate a new command buffer
3. Reset the command pool the command buffer was allocated from (which resets *all* of the command buffers allocated from the command pool) and record new commands to the command buffer

1. 重置指令缓冲（这会清除其中已经记录的指令），然后向指令缓冲中记录新的指令
2. 释放指令缓冲（这会将内存返还给指令池），然后重新分配一个指令缓冲
3. 重置指令池（这会重置*所有*从这个指令池中分配的指令缓冲），然后向指令缓冲中记录新的指令

<!-- 我怎么感觉这三种方法都太极端了？ -->

Let's look at what would be required to implement each of these approaches.

让我们看看实现这三种方法分别需要做什么。

### 1. 重置指令缓冲

By default, command buffers cannot be reset and are effectively immutable once they have been recorded. The ability to reset them is an option that must be enabled on our command pool during its creation and will be applied to any command buffers allocated from this command pool. Add the `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` flag to the create info structure for the command pool in `^create_command_pool`.

默认情况下，指令缓冲是不可重置的，一旦记录了指令，它们就是不可变的。重置指令缓冲的能力是一个选项，必须在创建指令池时启用，这个选项会应用到从这个指令池分配的所有指令缓冲。在 `create_command_pool` 中，为指令池创建信息添加 `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` 标志。

```rust,noplaypen
let info = vk::CommandPoolCreateInfo::builder()
    .flags(vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER)
    .queue_family_index(indices.graphics);

data.command_pool = device.create_command_pool(&info, None)?;
```

Next, create a new method for the `App` struct, `update_command_buffer`. This method will be called each frame to reset and rerecord the command buffer for the framebuffer that will be used for the current frame.

接着，为 `App` 结构体创建一个新的方法 `update_command_buffer`。这个方法会在每一帧被调用，用来重置并重新记录当前帧使用的帧缓冲的指令缓冲。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    Ok(())
}
```

Call the new method from the `render` method right before the uniform buffers for the frame are updated (or after, the order of these two statements is not important).

在 `render` 方法中，在更新帧的 uniform 缓冲之前（或者之后，这两个语句的顺序不重要）调用这个新方法。

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    self.update_command_buffer(image_index)?;
    self.update_uniform_buffer(image_index)?;

    // ...
}
```

Note that we do need to be careful about when we call `update_command_buffer`. This method will reset the command buffer which could cause serious issues if the command buffer is still being used to render a previously submitted frame. This issue was also discussed in the [`Descriptor set layout and buffer` chapter](../uniform/descriptor_set_layout_and_buffer.html#updating-uniform-data) which is why the call to `App::update_uniform_buffer` is where it is. As discussed in more detail in that chapter, both of these calls only happen after the call to `wait_for_fences` which waits for the GPU to be done with the acquired swapchain image and its associated resources so we are safe to do whatever we want with the command buffer.

我们要注意，在调用 `update_command_buffer` 时要多加小心。如果一个指令缓冲仍然在渲染一个之前已经提交的帧，重置这个指令缓冲会引发严重的问题。这个问题在 [`Descriptor set layout and buffer` 章节](../uniform/descriptor_set_layout_and_buffer.html#updating-uniform-data)中也有讨论，这也是为什么我们在这里调用 `App::update_uniform_buffer`。正如在那一章中详细讨论的那样，这两个函数都是在调用 `wait_for_fences` 之后被调用的，`wait_for_fences` 会等待 GPU 用完获取到的交换链图像及其相关资源，所以我们可以放心地对指令缓冲做任何事情。

In the new method, reset the command buffer with `reset_command_buffer`.

在这个新方法中，调用 `reset_command_buffer` 来重置指令缓冲。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    let command_buffer = self.data.command_buffers[image_index];

    self.device.reset_command_buffer(
        command_buffer,
        vk::CommandBufferResetFlags::empty(),
    )?;

    Ok(())
}
```

Once `reset_command_buffer` has returned, the command buffer will be reset to its initial state, no different than a new command buffer freshly allocated from a command pool.

一旦 `reset_command_buffer` 返回，指令缓冲就会被重置回初始状态，和从指令池中新分配的指令缓冲没有什么区别。

Now we can move the command buffer recording code out of `create_command_buffers` and into `update_command_buffer`. The loop over the command buffers is no longer necessary since we are only recording one command buffer per frame. Other than that, only a few mechanical changes are needed to migrate this code to our new method (e.g., replacing references to the loop counter `i` with `image_index`).

现在我们可以将记录指令缓冲的代码从 `create_command_buffers` 移动到 `update_command_buffer` 中。我们不需要再循环遍历指令缓冲了，因为我们每一帧只记录一个指令缓冲。除此之外，只需要做一些机械的修改就可以将这段代码迁移到我们的新方法中（例如，将对循环计数器 `i` 的引用替换为 `image_index`）。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    // ...

    let model = Mat4::from_axis_angle(
        vec3(0.0, 0.0, 1.0),
        Deg(0.0)
    );

    let model_bytes = &*slice_from_raw_parts(
        &model as *const Mat4 as *const u8,
        size_of::<Mat4>()
    );

    let info = vk::CommandBufferBeginInfo::builder();

    self.device.begin_command_buffer(command_buffer, &info)?;

    let render_area = vk::Rect2D::builder()
        .offset(vk::Offset2D::default())
        .extent(self.data.swapchain_extent);

    let color_clear_value = vk::ClearValue {
        color: vk::ClearColorValue {
            float32: [0.0, 0.0, 0.0, 1.0],
        },
    };

    let depth_clear_value = vk::ClearValue {
        depth_stencil: vk::ClearDepthStencilValue { depth: 1.0, stencil: 0 },
    };

    let clear_values = &[color_clear_value, depth_clear_value];
    let info = vk::RenderPassBeginInfo::builder()
        .render_pass(self.data.render_pass)
        .framebuffer(self.data.framebuffers[image_index])
        .render_area(render_area)
        .clear_values(clear_values);

    self.device.cmd_begin_render_pass(command_buffer, &info, vk::SubpassContents::INLINE);
    self.device.cmd_bind_pipeline(command_buffer, vk::PipelineBindPoint::GRAPHICS, self.data.pipeline);
    self.device.cmd_bind_vertex_buffers(command_buffer, 0, &[self.data.vertex_buffer], &[0]);
    self.device.cmd_bind_index_buffer(command_buffer, self.data.index_buffer, 0, vk::IndexType::UINT32);
    self.device.cmd_bind_descriptor_sets(
        command_buffer,
        vk::PipelineBindPoint::GRAPHICS,
        self.data.pipeline_layout,
        0,
        &[self.data.descriptor_sets[image_index]],
        &[],
    );
    self.device.cmd_push_constants(
        command_buffer,
        self.data.pipeline_layout,
        vk::ShaderStageFlags::VERTEX,
        0,
        model_bytes,
    );
    self.device.cmd_push_constants(
        command_buffer,
        self.data.pipeline_layout,
        vk::ShaderStageFlags::FRAGMENT,
        64,
        &0.25f32.to_ne_bytes()[..],
    );
    self.device.cmd_draw_indexed(command_buffer, self.data.indices.len() as u32, 1, 0, 0, 0);
    self.device.cmd_end_render_pass(command_buffer);

    self.device.end_command_buffer(command_buffer)?;

    Ok(())
}
```

With these changes in place, our program can now execute different rendering commands every frame which permits dynamic scenes! Let's exercise this new capability by restoring the rotation of the model to its former glory. Replace the model matrix calculation in `App::update_command_buffer` with the old calculation that rotates the model over time.

有了这些修改，我们的程序现在可以每一帧执行不同的渲染指令，这样就可以实现动态场景了！让我们通过将模型矩阵的计算恢复到旧的状态来练习这个新功能。在 `App::update_command_buffer` 中，将模型矩阵的计算替换为旧的计算，这样就可以让模型随着时间旋转。

```rust,noplaypen
let time = self.start.elapsed().as_secs_f32();

let model = Mat4::from_axis_angle(
    vec3(0.0, 0.0, 1.0),
    Deg(0.0) * time
);

let model_bytes = &*slice_from_raw_parts(
    &model as *const Mat4 as *const u8,
    size_of::<Mat4>()
);
```

Run the program to see that the model should now be back to rotating now that we are pushing an updated model matrix to the shaders every frame.

运行程序，你会发现模型现在又开始旋转了，因为我们每一帧都向着色器推送了一个更新的模型矩阵。

![](../images/spinning_ghost_model.png)

Lastly, since we are now only submitting our command buffers once before resetting them, we should let Vulkan know this so it can better understand the behavior of our program. This is accomplished by passing the `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` flag when starting to record a command buffer.

最后，由于我们现在只在重置指令缓冲之前提交一次指令缓冲，我们应该让 Vulkan 知道这一点，这样它就可以更好地理解我们程序的行为。在开始记录指令缓冲时，传递 `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` 标志就可以实现这一点。

```rust,noplaypen
let info = vk::CommandBufferBeginInfo::builder()
    .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);

self.device.begin_command_buffer(command_buffer, &info)?;
```

You might recall we've used this flag before, there should already be a usage of this flag in the `begin_single_time_commands` function. This flag isn't required by Vulkan for correctness if you are only using command buffers once before resetting or freeing them, but this knowledge may allow the Vulkan driver to better optimize its handling of our single-use command buffers.

你可能还记得我们之前在 `begin_single_time_commands` 函数中使用过这个标志。Vulkan 并不强制要求使用这个标志，但如果你只在重置或释放指令缓冲之前使用一次指令缓冲，有了这个标志提供的信息，Vulkan 驱动或许能更好地优化对一次性指令缓冲的处理。

### 2. 重新分配指令缓冲

Next we'll take a look at allocating new command buffers each frame.

接着我们来看看每一帧都重新分配指令缓冲要怎么做。

Replace the code used to reset the command buffer at the beginning of `update_command_buffer` with code that replaces the previous command buffer with a new command buffer.

移除我们刚才在 `update_command_buffer` 开头重置指令缓冲的代码。添加以下代码，用新的指令缓冲替换旧的指令缓冲：

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    let allocate_info = vk::CommandBufferAllocateInfo::builder()
        .command_pool(self.data.command_pool)
        .level(vk::CommandBufferLevel::PRIMARY)
        .command_buffer_count(1);

    let command_buffer = self.device.allocate_command_buffers(&allocate_info)?[0];
    self.data.command_buffers[image_index] = command_buffer;

    // ...
}
```

You could now run the program and see that the program works exactly like it did before, but if you do don't leave it running for too long! You may have already noticed that we aren't freeing the previous command buffer before we allocate a new one. If you observe the memory usage of our program after this change you'll see the memory usage start rising alarmingly fast as we rapidly collect thousands of derelict command buffers that are never recycled.

现在你可以运行程序，你会发现程序的运行效果和之前完全一样，但是如果你让它多运行一会，就会发现问题了！你可能已经注意到了，在分配新的指令缓冲之前，我们并没有释放之前的指令缓冲。如果你观察这个修改后的程序的内存使用情况，你会发现内存使用量迅速上升，因为很快就会出现数千个被遗弃的指令缓冲，而这些指令缓冲永远不会被回收。

Return the memory used by the previous command buffer to the command pool by freeing it at the beginning of `update_command_buffer`.

在 `update_command_buffer` 的开头释放原先的指令缓冲，将其使用的内存返还给指令池。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    let previous = self.data.command_buffers[image_index];
    self.device.free_command_buffers(self.data.command_pool, &[previous]);

    // ...
}
```

Now when you run the program you should see stable memory usage instead of the program trying to gobble up all of the RAM on your system as if it thinks it's an Electron application.

现在运行程序，你会发现内存使用量稳定了下来，而不是像 Electron 应用一样试图吞噬系统上所有的内存。

We no longer need the `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` flag for our command pool since we aren't resetting command pools any more. Leaving this flag wouldn't affect the correctness of our program, but it could have a negative performance impact since it forces the command pool to allocate command buffers in such a way that they are resettable.

现在我们不再需要 `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER` 标志了，因为我们不再重置指令池了。保留这个标志不会影响我们程序的正确性，但是它可能会对性能产生负面影响，因为它会强制指令池以可以重置的方式分配指令缓冲。

We'll replace this flag with `vk::CommandPoolCreateFlags::TRANSIENT` which tells Vulkan that the command buffers we'll be allocating with this command pool will be "transient", i.e. short-lived.

我们把标志换成 `vk::CommandPoolCreateFlags::TRANSIENT`，这会告诉 Vulkan，我们将使用这个指令池分配的指令缓冲是“稍纵即逝的”，也就是说，这些指令缓冲会非常短命。

```rust,noplaypen
let info = vk::CommandPoolCreateInfo::builder()
    .flags(vk::CommandPoolCreateFlags::TRANSIENT)
    .queue_family_index(indices.graphics);

data.command_pool = device.create_command_pool(&info, None)?;
```

Like `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT`, this flag does not affect the correctness of our program but it may allow the Vulkan driver to better optimize the handling of our short-lived command buffers.

和 `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` 一样，这个标志不会影响我们程序的正确性，但是它可能会让 Vulkan 驱动更好地优化对短命的指令缓冲的处理。

### 3. 重置指令池

Next we'll look at resetting our entire command pool which will reset all of our active command buffers in one fell swoop.

接着我们来看看重置整个指令池，这会一举重置所有活动的指令缓冲。

However, we immediately run into a problem with this approach. We can't reset *all* of our command buffers each frame because some of them might still be in use! The `wait_for_fences` call in `App::render` ensures that we are safe to reset the command buffer associated with the current framebuffer, but there might be other command buffers still in use.

然而，我们立刻就遇到了一个问题。我们不能每一帧都重置*所有*的指令缓冲，因为有些指令缓冲可能仍然在使用中！`App::render` 中的 `wait_for_fences` 调用确保我们可以安全地重置当前帧缓冲的指令缓冲，但是可能还有其他指令缓冲仍然在使用中。

We could continue down this path, but it would prevent our program from having multiple frames in-flight concurrently. This ability is important to maintain because, as discussed back in the [`Rendering and presentation` chapter](../drawing/rendering_and_presentation.html#frames-in-flight), it allows us to better leverage our hardware since the CPU will spend less time waiting on the GPU and vice-versa.

我们可以继续沿着这条路走下去，但这样的话我们的程序就不能并行渲染多个帧了。保持多帧并行渲染的能力很重要，因为正如在[渲染与呈现](../drawing/rendering_and_presentation.html#frames-in-flight)那一章提到的，这可以让我们更好地利用硬件：CPU 会花费更少的时间等待 GPU，反之亦然。

Instead, we will alter our program to maintain a separate command pool for each framebuffer. This way we can freely reset the command pool associated with the current framebuffer without worrying about breaking any previously submitted frames that are still in-flight.

我们会选择另一种方式，为每个帧缓冲维护一个单独的指令池。这样我们就可以自由地重置当前帧缓冲关联的指令池，而不用担心会破坏任何仍在渲染中的之前提交的帧。

You might think that this is overkill, why maintain separate command pools just so we can reset command buffers one at a time? Wouldn't it be simpler, and probably even faster, to continue freeing or resetting our command buffers each frame? Is this just a pedagogical exercise? Is the author of this tutorial a fraud?

你可能会觉得这有点*高射炮打蚊子*，为什么仅仅为了每次只重置一个指令缓冲，就要创建多个单独的指令池呢？在每一帧中释放或者重置指令缓冲不是更简单，而且或许还能更快吗？这只是一个教学练习吗？本教程的作者是个骗子吗？

To put these questions on hold for a bit (well maybe not the last one), a sneak preview of the next chapter is that it will involve managing multiple command buffers per frame rather than the single command buffer per frame we've been working with so far. Then it will become simpler, and probably faster, to deallocate all of these command buffers in one go by resetting the command pool instead of deallocating them individually.

为了让你能够放心地暂时搁置这些问题（好吧，也许不包括最后一个问题），让我们来预览一下下一章的内容。下一章将涉及每一帧管理多个指令缓冲，而不是我们到目前为止一直在使用的每一帧一个指令缓冲。那么，通过重置指令池一次性释放所有这些指令缓冲，程序将变得更简单，而且或许还会更快。

We are going to leave the current command pool in place since it will be used for allocating command buffers during initialization. Add a field to `AppData` to hold one command pool per framebuffer and rename the existing `^create_command_pool` function to `create_command_pools` to reflect its increased responsibilities.

我们将保留现有的这个指令池，因为它会在程序初始化期间被用来分配指令缓冲。在 `AppData` 中添加一个字段，为每个帧缓冲保存一个指令池，并将现有的 `create_command_pool` 函数重命名为 `create_command_pools`，以反映它新增加的职能。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_command_pools(&instance, &device, &mut data)?;
        // ...
    }
}

struct AppData {
    // ...
    command_pools: Vec<vk::CommandPool>,
    command_buffers: Vec<vk::CommandBuffer>,
    // ...
}

unsafe fn create_command_pools(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    // ...
}
```

Create a new `^create_command_pool` function which will be used to create a command pool for short-lived command buffers that can be submitted to graphics queues.

创建一个新的 `create_command_pool` 函数，用来创建一个指令池，这个指令池会被用于创建可以提交到图形队列的、短命的指令缓冲。

```rust,noplaypen
unsafe fn create_command_pool(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<vk::CommandPool> {
    let indices = QueueFamilyIndices::get(instance, data, data.physical_device)?;

    let info = vk::CommandPoolCreateInfo::builder()
        .flags(vk::CommandPoolCreateFlags::TRANSIENT)
        .queue_family_index(indices.graphics);

    Ok(device.create_command_pool(&info, None)?)
}
```

With this function available, we can easily update `create_command_pools` to create both our existing global command pool and the new per-framebuffer command pools.

有了这个函数，我们可以很容易地更新 `create_command_pools`，来创建我们现有的全局指令池和新的每帧指令池。

```rust,noplaypen
unsafe fn create_command_pools(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    data.command_pool = create_command_pool(instance, device, data)?;

    let num_images = data.swapchain_images.len();
    for _ in 0..num_images {
        let command_pool = create_command_pool(instance, device, data)?;
        data.command_pools.push(command_pool);
    }

    Ok(())
}
```

Now we need to create the command buffers using these new per-framebuffer command pools. Update `create_command_buffers` to use a separate call to `allocate_command_buffers` for each command buffer so that each can be associated with one of the per-framebuffer command pools.

现在我们需要使用这些新的每帧指令池来创建指令缓冲。更新 `create_command_buffers`，为每个指令缓冲使用一个单独的 `allocate_command_buffers` 调用，这样每个指令缓冲就可以关联到一个每帧指令池。

```rust,noplaypen
unsafe fn create_command_buffers(device: &Device, data: &mut AppData) -> Result<()> {
    let num_images = data.swapchain_images.len();
    for image_index in 0..num_images {
        let allocate_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(data.command_pools[image_index])
            .level(vk::CommandBufferLevel::PRIMARY)
            .command_buffer_count(1);

        let command_buffer = device.allocate_command_buffers(&allocate_info)?[0];
        data.command_buffers.push(command_buffer);
    }

    Ok(())
}
```

Update `App::update_command_buffer` to reset the per-framebuffer command pool instead of freeing and reallocating the command buffer. This will also reset any command buffers created with this command pool so we don't need to do anything else to be able to reuse the command buffer.

更新 `App::update_command_buffer`，重置每帧指令池，而不是释放并重新分配指令缓冲。这也会重置使用这个指令池创建的任何指令缓冲，所以我们不需要做任何其他事情，就可以重用指令缓冲。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    let command_pool = self.data.command_pools[image_index];
    self.device.reset_command_pool(command_pool, vk::CommandPoolResetFlags::empty())?;

    let command_buffer = self.data.command_buffers[image_index];

    // ...
}
```

Run the program now and make sure that our new command buffer recycling strategy still produces the same result as before. If you have the validation layer enabled, you will be reminded while the program is shutting down that we are not cleaning up these new command pools. Update `App::destroy` to destroy them.

现在运行程序，确保新的指令缓冲重用策略仍然会产生和之前一样的结果。如果你启用了校验层，当程序关闭时，校验层会提醒我们没有清理这些新的指令池。更新 `App::destroy`，销毁它们。

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.data.command_pools
        .iter()
        .for_each(|p| self.device.destroy_command_pool(*p, None));
    // ...
}
```

Finally, delete the call to `free_command_buffers` in `App::destroy_swapchain`. This call now incorrectly attempts to return the memory assigned to the per-framebuffer command buffers to the global command pool despite the fact that these command buffers are no longer allocated from this command pool. Leaving this code in will most likely result in our program crashing when resizing the window or otherwise forcing a recreation of the swapchain. We no longer need to manage the deletion of individual command buffers since we are now managing this at the command pool level.

最后，在 `App::destroy_swapchain` 中删除对 `free_command_buffers` 的调用。这个调用会错误地尝试将分配给每帧指令缓冲的内存返还给全局指令池，而现在这些指令缓冲并不是从全局指令池指令池分配的。如果保留这段代码，当窗口大小改变或者强制重新创建交换链时，我们的程序很可能会崩溃。我们不再需要管理单个指令缓冲的删除，因为我们现在是在指令池级别管理这个。

## 结论

We've now explored the basic approaches Vulkan offers for recycling command buffers so that we can change the commands our program submits dynamically, whether in response to user input or to some other signal. These approaches can be mixed in any way you could imagine, demonstrating the power and flexibility Vulkan grants to programmers.

现在我们已经探索了 Vulkan 提供的重用指令缓冲的基本方式，现在我们可以动态地改变程序提交的指令，无论是为了响应用户输入还是响应其他信号。这些方法可以以任何你能想象的方式被组合使用，这也恰恰展示了 Vulkan 赋予程序员的强大和灵活性。

If you are feeling a bit overwhelmed about all the possible ways you could go about architecting a Vulkan program with respect to command pools and command buffers, don't worry! The next chapter is going to make things even more complicated.

如果你对于如何在指令池和指令缓冲方面设计 Vulkan 程序感到有点不知所措，不要担心！下一章会让事情变得更加复杂。
