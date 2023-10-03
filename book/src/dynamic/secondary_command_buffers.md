# 次级指令缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/dynamic/secondary_command_buffers.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>The previous chapters of this tutorial that are not marked by this disclaimer were directly adapted from <https://github.com/Overv/VulkanTutorial>.<br/><br/>This chapter and the following chapters are instead original creations from someone who is most decidedly not an expert in Vulkan. An authoritative tone has been maintained, but these chapters should be considered a "best effort" by someone still learning Vulkan.<br/><br/>If you have questions, suggestions, or corrections, please [open an issue](https://github.com/KyleMayes/vulkanalia/issues)!

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>前面没有这个声明的章节都是直接从 <https://github.com/Overv/VulkanTutorial> 改编而来。<br/><br/>这一章和后面的章节都是原创，作者并不是 Vulkan 的专家。作者尽力保持了权威的语气，但是这些章节应该被视为一个 Vulkan 初学者的“尽力而为”。<br/><br/>如果你有问题、建议或者修正，请[提交 issue](https://github.com/KyleMayes/vulkanalia/issues)！

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/32_secondary_command_buffers.rs)

While our program now submits different commands to be executed every frame, we still haven't quite accomplished our original goal of changing *what* our program renders dynamically. In this chapter we'll alter our program to support rendering between 1 and 4 instances of the model in response to user input.

现在我们的程序会在每一帧提交不同的命令，但是我们还没有达到我们的最初目标：动态地改变程序渲染的内容。在这一章中，我们将修改我们的程序，使其能够根据用户输入渲染 1 到 4 个模型实例。

We'll accomplish this using *secondary command buffers*, a Vulkan feature that allows us to build re-usable sequences of commands and then execute those commands from *primary command buffers*. Secondary command buffers aren't at all necessary to implement this change, but our first time rendering multiple things is a good time to introduce them.

我们将通过*次级指令缓冲*来实现这个功能。*次级指令缓冲*是一个 Vulkan 特性，可以用来构建可重用的命令序列，然后从*主指令缓冲*中执行这些命令。次级指令缓冲并不是实现这个功能的必要条件，但是我们第一次渲染多个物体的时候，正好可以介绍一下它。

## 主指令缓冲 vs 次级指令缓冲

All of the command buffers we've used thus far have been primary command buffers, meaning they can be submitted directly to a Vulkan queue to be executed by the device. Secondary command buffers are instead executed indirectly by being called from primary command buffers and may not be submitted to queues.

目前为止我们只用过主指令缓冲，也就是可以被直接提交到 Vulkan 队列并在设备上执行的指令缓冲。次级指令缓冲则不能被提交到队列，而是会被主指令缓冲调用并间接地执行。

The usage of secondary command buffers offers two primary advantages:

使用次级指令缓冲有两个主要的优点：

1. Secondary command buffers may be allocated and recorded in parallel which allows you to better leverage modern hardware with its panoply of CPU cores

2. The lifetime of secondary command buffers can managed independently of one another so you can have a mixture of long-lived or permanent secondary command buffers that intermingle with frequently updated secondary command buffers which allows you to reduce the number of command buffers you need to create every frame

1. 次级指令缓冲可以被并行地分配和记录，这样你就可以更好地利用现代硬件的众多 CPU 核心
2. 次级指令缓冲的生存期可以独立地管理，这样你就可以同时拥有长期或永久的次级指令缓冲和经常更新的次级指令缓冲，从而减少每一帧需要创建的指令缓冲的数量

Both of these points are true for primary command buffers as well, but primary command buffers have a significant limitation that effectively prevents them from fulfilling these use cases. Multiple primary command buffers may not be executed within the same render pass instance meaning that if you wanted to execute multiple primary command buffers for a frame, each primary command buffer would need to start with `cmd_begin_render_pass` and end with `cmd_end_render_pass`.

这两点对主指令缓冲也是成立的，但是主指令缓冲有一个重大的限制，导致它不能利用这些优势。多个主指令缓冲无法在同一个渲染流程中执行，也就是说如果你想在一帧中执行多个主指令缓冲，每个主指令缓冲都需要以 `cmd_begin_render_pass` 开始，以 `cmd_end_render_pass` 结束。

This might not sound like a big deal but beginning a render pass instance can be a pretty heavyweight operation and needing to do this many times per frame can destroy performance on some hardware. Secondary command buffers avoid this problem by being able to inherit the render pass instance as well as other state from the primary command buffer it is called from.

这听起来不是什么大问题，但是开始一个渲染流程实例是一个非常耗费资源的操作，而且如果每一帧都需要多次开始渲染流程，那么性能就会在某些硬件上大幅下降。次级指令缓冲可以从调用它的主指令缓冲那里继承渲染流程实例以及其他状态，从而避免了这个问题。

## 多个模型实例

Let's get started by adding a field to `AppData` that will contain our new secondary command buffers. We will have multiple secondary command buffers per frame, one for each model instance we are rendering, so this will be a list of lists.

我们从给 `AppData` 添加一个字段开始，这个字段将包含我们新的次级指令缓冲。我们每一帧都会有多个次级指令缓冲，每个次级指令缓冲都对应一个我们正在渲染的模型实例，所以这将是一个列表的列表。

```rust,noplaypen
struct AppData {
    // ...
    command_buffers: Vec<vk::CommandBuffer>,
    secondary_command_buffers: Vec<Vec<vk::CommandBuffer>>,
    // ...
}
```

In an application more realistic than the one we are building, the number of secondary command buffers we need to render a frame might vary significantly over time. In addition, we likely wouldn't know the maximum number of secondary command buffers the application needs ahead of time.

在真实的应用程序中，我们在一帧中需要渲染的次级指令缓冲的数量可能会随着时间的推移而显著变化。此外，我们可能不会提前知道应用程序最多需要多少个次级指令缓冲。

We do know the maximum in this case, but we will pretend we don't and adopt an approach closer to what a real-world application would. Instead of allocating secondary command buffers during initialization like we allocate primary command buffers, we will allocate secondary command buffers on-demand. We'll still need to populate the outer `Vec` with empty lists of secondary command buffers so update `create_command_buffers` to accomplish this.

在这个例子里我们其实是知道最大值的，但我们假装不知道，并且采取一个更接近真实应用程序的方法。我们不会像分配主指令缓冲那样在初始化时分配次级指令缓冲，而是在需要时分配次级指令缓冲。我们仍然需要用空的次级指令缓冲列表填充外部的 `Vec`，所以我们需要更新 `create_command_buffers` 来实现这一点。

```rust,noplaypen
unsafe fn create_command_buffers(device: &Device, data: &mut AppData) -> Result<()> {
    // ...

    data.secondary_command_buffers = vec![vec![]; data.swapchain_images.len()];

    Ok(())
}
```

Add a new method for the `App` struct called `update_secondary_command_buffer` that we'll use to allocate (if necessary) and record a secondary command buffer for one of the 4 model instances we will be rendering. The `model_index` parameter indicates which of the 4 model instances the secondary command buffer should render.

为 `App` 新增一个 `update_secondary_command_buffer` 方法，我们将会用这个方法来（在需要时）为我们将要渲染的 4 个模型实例之一分配并记录次级指令缓冲。`model_index` 参数表示次级指令缓冲应该渲染的 4 个模型实例中的哪一个。

```rust,noplaypen
unsafe fn update_secondary_command_buffer(
    &mut self,
    image_index: usize,
    model_index: usize,
) -> Result<vk::CommandBuffer> {
    self.data.secondary_command_buffers.resize_with(image_index + 1, Vec::new);
    let command_buffers = &mut self.data.secondary_command_buffers[image_index];
    while model_index >= command_buffers.len() {
        let allocate_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(self.data.command_pools[image_index])
            .level(vk::CommandBufferLevel::SECONDARY)
            .command_buffer_count(1);

        let command_buffer = self.device.allocate_command_buffers(&allocate_info)?[0];
        command_buffers.push(command_buffer);
    }

    let command_buffer = command_buffers[model_index];

    let info = vk::CommandBufferBeginInfo::builder();

    self.device.begin_command_buffer(command_buffer, &info)?;

    self.device.end_command_buffer(command_buffer)?;

    Ok(command_buffer)
}
```

This code will allocate secondary command buffers for the model instances as they are needed but will reuse them after their initial allocation. Like with the primary command buffers, we can freely use any previously allocated secondary command buffers because we are resetting the command pool they were allocated with.

这些代码将会为模型实例在需要时分配次级指令缓冲，但是在初次分配后会重用它们。和主指令缓冲一样，我们可以自由地使用任何之前分配的次级指令缓冲，因为我们正在重置它们所分配的指令池。

Before we continue, we need to provide some additional information to Vulkan that is unique to secondary command buffers before recording this command buffer. Create an instance of `vk::CommandBufferInheritanceInfo` that specifies the render pass, subpass index, and framebuffer the secondary command buffer will be used in conjunction with and then provide that inheritance info to `begin_command_buffer`.

在开始记录次级指令缓冲之前，我们需要向 Vulkan 提供一些次级指令缓冲特有的额外信息。创建一个 `vk::CommandBufferInheritanceInfo` 实例，指定将与次级指令缓冲一起使用的渲染流程、子流程索引和帧缓冲，然后将这个继承信息提供给 `begin_command_buffer`。

```rust,noplaypen
let inheritance_info = vk::CommandBufferInheritanceInfo::builder()
    .render_pass(self.data.render_pass)
    .subpass(0)
    .framebuffer(self.data.framebuffers[image_index]);

let info = vk::CommandBufferBeginInfo::builder()
    .inheritance_info(&inheritance_info);

self.device.begin_command_buffer(command_buffer, &info)?;
```

As mentioned previously, secondary command buffers can inherit some state from the primary command buffers they are executed from. This inheritance info describes the command buffer state the secondary command buffer will be compatible with and may validly inherit.

正如我们之前所提到的，次级指令缓冲可以从执行它的主指令缓冲那里继承一些状态。这个继承信息描述了次级指令缓冲将与之兼容并且可以合法继承的指令缓冲状态。

The render pass and subpass index are *required* to inherit that state, but the framebuffer is only specified here as a potential performance boost. You may omit it, but Vulkan may be able to better optimize the secondary command buffer to render to the specified framebuffer.

渲染流程和子流程索引是*必填*的，但是帧缓冲只是作为潜在的性能提升在这里指定。你可以省略它，但是 Vulkan 可能能够更好地优化次级指令缓冲以渲染到指定的帧缓冲。

This isn't enough to actually inherit the render pass, we need to also provide `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE` to `begin_command_buffer`. This tells Vulkan that this secondary command buffer will be executed entirely inside a render pass.

这还不足以实际地继承渲染流程，我们还需要将 `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE` 提供给 `begin_command_buffer`。这告诉 Vulkan 这个次级指令缓冲将完全在渲染流程内执行。

```rust,noplaypen
let info = vk::CommandBufferBeginInfo::builder()
    .flags(vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE)
    .inheritance_info(&inheritance_info);

self.device.begin_command_buffer(command_buffer, &info)?;
```

With inheritance set up, move the code that calculates the push constant values out of `App::update_command_buffer` and into `App::update_secondary_command_buffer` after the secondary command buffer is allocated. While you're at it, have the opacity of the model instance depend on the model index to add some variety to our scene, ranging from 25% to 100%.

将计算推送常量值的代码从 `App::update_command_buffer` 移动到 `App::update_secondary_command_buffer`，放在次级指令缓冲分配之后。在此之际，让模型实例的不透明度取决于模型索引，以增加我们场景的多样性，范围从 25% 到 100%。

```rust,noplaypen
unsafe fn update_secondary_command_buffer(
    &mut self,
    image_index: usize,
    model_index: usize,
) -> Result<vk::CommandBuffer> {
    // ...

    let command_buffer = self.device.allocate_command_buffers(&allocate_info)?[0];

    let time = self.start.elapsed().as_secs_f32();

    let model = Mat4::from_axis_angle(
        vec3(0.0, 0.0, 1.0),
        Deg(0.0) * time
    );

    let model_bytes = &*slice_from_raw_parts(
        &model as *const Mat4 as *const u8,
        size_of::<Mat4>()
    );

    let opacity = (model_index + 1) as f32 * 0.25;
    let opacity_bytes = &opacity.to_ne_bytes()[..];

    // ...
}
```

Next we are going to move the rendering commands out of the primary command buffer and into the secondary command buffer. The primary command buffer will still be used to begin and end the render pass instance since it will be inherited by our secondary command buffers, but all of the commands in `App::update_command_buffer` between (but not including) `cmd_begin_render_pass` and `cmd_end_render_pass` should be moved into `App::update_secondary_command_buffer`.

接下来我们将把渲染命令从主指令缓冲移动到次级指令缓冲。主指令缓冲仍然会用于开始和结束渲染流程实例，因为它将被我们的次级指令缓冲继承，但是 `App::update_command_buffer` 中在 `cmd_begin_render_pass` 和 `cmd_end_render_pass` 之间（但不包括这两个命令）的所有命令都应该被移动到 `App::update_secondary_command_buffer`。

```rust,noplaypen
unsafe fn update_secondary_command_buffer(
    &mut self,
    image_index: usize,
    model_index: usize,
) -> Result<vk::CommandBuffer> {
    // ...

    self.device.begin_command_buffer(command_buffer, &info)?;

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
        opacity_bytes,
    );
    self.device.cmd_draw_indexed(command_buffer, self.data.indices.len() as u32, 1, 0, 0, 0);

    self.device.end_command_buffer(command_buffer)?;

    // ...
}
```

Now that we can easily create secondary command buffers for rendering the model instance, call our new method in `App::update_command_buffers` and execute the returned secondary command buffer using `cmd_execute_commands`.

现在我们可以轻松地创建用于渲染模型实例的次级指令缓冲，在 `App::update_command_buffers` 中调用我们的新方法，并使用 `cmd_execute_commands` 执行返回的次级指令缓冲。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    // ...

    self.device.cmd_begin_render_pass(command_buffer, &info, vk::SubpassContents::INLINE);

    let secondary_command_buffer = self.update_secondary_command_buffer(image_index, 0)?;
    self.device.cmd_execute_commands(command_buffer, &[secondary_command_buffer]);

    self.device.cmd_end_render_pass(command_buffer);

    // ...
}
```

This change has invalidated our call to `cmd_begin_render_pass` because we are providing `vk::SubpassContents::INLINE` which indicates we will be recording rendering commands directly into the primary command buffer. Now that we've moved the rendering commands into the secondary command buffer, we need to use `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS`.

这个改变使我们的 `cmd_begin_render_pass` 调用失效了，因为我们提供了 `vk::SubpassContents::INLINE`，这表示我们将直接将渲染命令记录到主指令缓冲中。现在我们已经将渲染命令移动到了次级指令缓冲中，所以我们需要使用 `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS`。

```rust,noplaypen
self.device.cmd_begin_render_pass(
    command_buffer,
    &info,
    vk::SubpassContents::SECONDARY_COMMAND_BUFFERS,
);
```

Note that these are mutually exclusive modes, you can't mix secondary command buffers and inline rendering commands in a render pass instance.

注意这是两种互斥的模式，你不能在渲染流程实例中混用次级指令缓冲和内联渲染命令。

If you run the program now, you should see the same ghostly model rotating exactly as it was before. Let's kick it up a notch by rendering 4 instances of the model by creating 4 secondary command buffers and executing them all from the primary command buffer.

如果你现在运行程序，你应该会看到和之前一样的幽灵模型在旋转。让我们通过创建 4 个次级指令缓冲并从主指令缓冲中执行它们来提高一下难度，渲染 4 个模型实例。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    // ...

    self.device.cmd_begin_render_pass(command_buffer, &info, vk::SubpassContents::INLINE);

    let secondary_command_buffers = (0..4)
        .map(|i| self.update_secondary_command_buffer(image_index, i))
        .collect::<Result<Vec<_>, _>>()?;
    self.device.cmd_execute_commands(command_buffer, &secondary_command_buffers[..]);

    self.device.cmd_end_render_pass(command_buffer);

    // ...
}
```

If you run the program again, you'll see a strange shimmering as the 4 model instances, being rendered at the same coordinates, experience a bad bout of [z-fighting](https://en.wikipedia.org/wiki/Z-fighting).

如果你再次运行程序，你会看到 4 个模型实例在同样的坐标上发生了[深度冲突](https://en.wikipedia.org/wiki/Z-fighting)，从而产生了奇怪的闪烁。

Update the model matrix calculation in `App::update_secondary_command_buffers` to translate the models before rotating them according to their model index.

修改 `App::update_secondary_command_buffers` 中计算模型矩阵的过程，使其在旋转模型之前根据模型索引将模型平移到不同的位置。

```rust,noplaypen
let y = (((model_index % 2) as f32) * 2.5) - 1.25;
let z = (((model_index / 2) as f32) * -2.0) + 1.0;

let time = self.start.elapsed().as_secs_f32();

let model = Mat4::from_translation(vec3(0.0, y, z)) * Mat4::from_axis_angle(
    vec3(0.0, 0.0, 1.0),
    Deg(90.0) * time
);
```

This code places the model instances in a grid on the Y and Z axes. However, due to the view matrix we're using, the camera is looking at this plane at 45 degree angles so let's update the view matrix in `App::update_uniform_buffer` to look directly at the YZ plane to better view our model instances.

这段代码将模型实例放置在 Y 轴和 Z 轴上的网格中。然而，由于我们使用的视图矩阵，相机是以 45 度的角度看着这个平面的，所以让我们在 `App::update_uniform_buffer` 中更新视图矩阵，使其直接看向 YZ 平面，以更好地查看我们的模型实例。

```rust,noplaypen
let view = Mat4::look_at_rh(
    point3(6.0, 0.0, 2.0),
    point3(0.0, 0.0, 0.0),
    vec3(0.0, 0.0, 1.0),
);
```

With a better vantage point secured, run the program and bask in its glory.

有了一个更好的视角，运行程序并享受它的荣耀吧。

![](../images/4_models.png)

Let's knock it up a notch with a blast from our spice weasel by allowing the user to determine how many of these models they want to render. Add a `models` field to the `App` struct and initialize it to 1 in the constructor.

让我们通过允许用户决定他们想要渲染多少个模型来提高一下难度。在构造函数中为 `App` 结构体添加一个 `models` 字段，并将其初始化为 1。

```rust,noplaypen
struct App {
    // ...
    models: usize,
}
```

Update the model index range in `App::update_command_buffer` to range from 0 to the value of the `models` field.

将 `App::update_command_buffer` 中的模型索引范围更新为从 0 到 `models` 字段的值。

```rust,noplaypen
let secondary_command_buffers = (0..self.models)
    .map(|i| self.update_secondary_command_buffer(image_index, i))
    .collect::<Result<Vec<_>, _>>()?;
```

Now that we have all this in place, we just need to increment and decrement the `models` field in response to user input. Start by importing the following `winit` types we'll need to handle keyboard input.

现在我们已经完成了所有的工作，我们只需要根据用户输入来增加和减少 `models` 字段。首先导入以下 `winit` 类型，我们将需要它们来处理键盘输入。

```rust,noplaypen
use winit::event::{ElementState, VirtualKeyCode};
```

Finally, add a case to the event match block in the `main` function that handles key presses and decrements `models` when the left arrow key is pressed (to a minimum of 1) and increments `models` when the right arrow key is pressed (to a maximum of 4).

最后，在 `main` 函数的事件匹配块中添加一个处理按键的分支，当按下左箭头键时，将 `models` 减少 1（最少为 1），当按下右箭头键时，将 `models` 增加 1（最多为 4）。

```rust,noplaypen
match event {
    // ...
    Event::WindowEvent { event: WindowEvent::KeyboardInput { input, .. }, .. } => {
        if input.state == ElementState::Pressed {
            match input.virtual_keycode {
                Some(VirtualKeyCode::Left) if app.models > 1 => app.models -= 1,
                Some(VirtualKeyCode::Right) if app.models < 4 => app.models += 1,
                _ => { }
            }
        }
    }
    // ...
}
```

Run the program and observe how the number of secondary command buffers we are allocating and executing each frame changes as you press the left and right arrow keys.

运行程序并观察，当你按下左右箭头键时，我们每一帧分配和执行的次级指令缓冲的数量会如何变化。

![](../images/3_models.png)

You should now be familiar with the basic tools you can use to efficiently render dynamic frames using Vulkan. There are many ways you can utilize these tools that each have different performance tradeoffs. Future tutorial chapters may explore this more in depth, but parallelizing the work of recording secondary command buffers using multiple threads is a common technique that usually results in significant performance wins on modern hardware.

现在你应该已经熟悉了使用 Vulkan 高效渲染动态帧的基本工具。你可以使用这些工具的许多方法，每种方法都有不同的性能权衡。未来的教程章节可能会更深入地探讨这个问题，但是使用多个线程并行地记录次级指令缓冲的工作是一种常见的技术，通常可以在现代硬件上获得显著的性能提升。
