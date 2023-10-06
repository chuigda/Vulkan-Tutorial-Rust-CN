# 次级指令缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/dynamic/secondary_command_buffers.html>
>
> Commit Hash: ceb4a3fc6d8ca565af4f8679c4889bcad7941338

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>前面没有这个声明的章节都是直接从 <https://github.com/Overv/VulkanTutorial> 改编而来。<br/><br/>这一章和后面的章节都是原创，作者并不是 Vulkan 的专家。作者尽力保持了权威的语气，但是这些章节应该被视为一个 Vulkan 初学者的“尽力而为”。<br/><br/>如果你有问题、建议或者修正，请[提交 issue](https://github.com/KyleMayes/vulkanalia/issues)！

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/32_secondary_command_buffers.rs)

现在我们的程序会在每一帧提交不同的指令，但是我们还没有达到我们的最初目标：动态地改变程序渲染的内容。在这一章中，我们将修改我们的程序，使其能够根据用户输入渲染 1 到 4 个模型实例。

我们将运用*次级指令缓冲*来实现这个功能。*次级指令缓冲*是一个 Vulkan 特性，可以用来构建可重用的指令序列，然后我们就可以从*主指令缓冲*中执行这些指令。实现这个功能并不是一定要使用次级指令缓冲，但是这是我们第一次渲染多个物体，正好可以介绍一下它。

## 主指令缓冲 vs 次级指令缓冲

目前为止我们只用过主指令缓冲，也就是可以被直接提交到 Vulkan 队列并在设备上执行的指令缓冲。次级指令缓冲则不能被提交到队列，而是会被主指令缓冲调用并间接地执行。

使用次级指令缓冲有两个主要的优点：

1. 次级指令缓冲可以被并行地分配和记录，这样你就可以更好地利用现代硬件的众多 CPU 核心
2. 次级指令缓冲的生存期可以独立地管理，这样你就可以同时拥有长期或永久的次级指令缓冲和经常更新的次级指令缓冲，从而减少每一帧需要创建的指令缓冲的数量

这两点对主指令缓冲也是成立的，但是主指令缓冲有一个重大的限制，导致它不能利用这些优势。多个主指令缓冲无法在同一个渲染流程中同时执行，也就是说如果你想在一帧中执行多个主指令缓冲，每个主指令缓冲都需要以 `cmd_begin_render_pass` 开始，以 `cmd_end_render_pass` 结束。

这听起来不是什么大问题，但是开始一个渲染流程实例是一个非常耗费资源的操作，而且如果每一帧都需要多次开始渲染流程，那么性能就会在某些硬件上大幅下降。次级指令缓冲可以从调用它的主指令缓冲那里继承渲染流程实例以及其他状态，从而避免了这个问题。

## 多个模型实例

我们从给 `AppData` 添加一个字段开始，这个字段将包含我们新的次级指令缓冲。我们每一帧都会有多个次级指令缓冲，每个次级指令缓冲都对应一个我们正在渲染的模型实例，所以这个字段是一个列表的列表。

```rust,noplaypen
struct AppData {
    // ...
    command_buffers: Vec<vk::CommandBuffer>,
    secondary_command_buffers: Vec<Vec<vk::CommandBuffer>>,
    // ...
}
```

在真实的应用程序中，我们在一帧中需要渲染的次级指令缓冲的数量可能会随着时间的推移而显著变化。此外，我们可能不会提前知道应用程序最多需要多少个次级指令缓冲。

在这个例子里我们其实是知道最大值的，但我们假装不知道，并且采取一个更接近真实应用程序的方法。我们不会像分配主指令缓冲那样在初始化时分配次级指令缓冲，而是在需要时分配次级指令缓冲。我们仍然需要用空的次级指令缓冲列表填充外部的 `Vec`，所以我们需要更新 `create_command_buffers` 来实现这一点。

```rust,noplaypen
unsafe fn create_command_buffers(device: &Device, data: &mut AppData) -> Result<()> {
    // ...

    data.secondary_command_buffers = vec![vec![]; data.swapchain_images.len()];

    Ok(())
}
```

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

这些代码将会为模型实例在需要时分配次级指令缓冲，但是在初次分配后会重用它们。和主指令缓冲一样，我们可以自由地使用任何之前分配的次级指令缓冲，因为我们正在重置它们所分配的指令池。

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

正如我们之前所提到的，次级指令缓冲可以从执行它的主指令缓冲那里继承一些状态。这个继承信息描述了次级指令缓冲将与哪些指令缓冲状态兼容，并合法地继承它们。

要继承指令缓冲状态，渲染流程和子流程索引是*必填项目*。而帧缓冲则是可选的，你可以省略它，但提供它的话 Vulkan 或许能够更好地优化次级指令缓冲。

这还不足以实际地继承渲染流程，我们还需要将 `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE` 提供给 `begin_command_buffer`。这告诉 Vulkan 这个次级指令缓冲将完全在渲染流程内执行。

```rust,noplaypen
let info = vk::CommandBufferBeginInfo::builder()
    .flags(vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE)
    .inheritance_info(&inheritance_info);

self.device.begin_command_buffer(command_buffer, &info)?;
```

将计算推送常量值的代码从 `App::update_command_buffer` 移动到 `App::update_secondary_command_buffer` 中分配次级指令缓冲之后。同时，让模型实例的不透明度取决于模型索引，范围从 25% 到 100%，以增加我们场景的多样性。

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
        Deg(90.0) * time
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

接下来我们将把渲染指令从主指令缓冲移动到次级指令缓冲。主指令缓冲仍然会用于开始和结束渲染流程实例，因为它将被我们的次级指令缓冲继承，但是 `App::update_command_buffer` 中在 `cmd_begin_render_pass` 和 `cmd_end_render_pass` 之间（但不包括这两个指令）的所有指令都应该被移动到 `App::update_secondary_command_buffer`。

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

这个改变使我们的 `cmd_begin_render_pass` 调用失效了，因为我们之前提供了 `vk::SubpassContents::INLINE`，这表示我们将直接将渲染指令记录到主指令缓冲中。现在我们已经将渲染指令移动到了次级指令缓冲中，因此我们需要改用 `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS`。

```rust,noplaypen
self.device.cmd_begin_render_pass(
    command_buffer,
    &info,
    vk::SubpassContents::SECONDARY_COMMAND_BUFFERS,
);
```

注意这是两种互斥的模式，你不能在渲染流程实例中混用次级指令缓冲和内联渲染指令。

如果你现在运行程序，你应该会看到和之前一样的幽灵模型在旋转。让我们通过创建 4 个次级指令缓冲并从主指令缓冲中执行它们来提高一下难度，渲染 4 个模型实例。

```rust,noplaypen
unsafe fn update_command_buffer(&mut self, image_index: usize) -> Result<()> {
    // ...

    self.device.cmd_begin_render_pass(command_buffer, &info, vk::SubpassContents::SECONDARY_COMMAND_BUFFERS);

    let secondary_command_buffers = (0..4)
        .map(|i| self.update_secondary_command_buffer(image_index, i))
        .collect::<Result<Vec<_>, _>>()?;
    self.device.cmd_execute_commands(command_buffer, &secondary_command_buffers[..]);

    self.device.cmd_end_render_pass(command_buffer);

    // ...
}
```

如果你再次运行程序，你会看到 4 个模型实例在同样的坐标上发生了[深度冲突](https://en.wikipedia.org/wiki/Z-fighting)，并因此产生了奇怪的闪烁。

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

这段代码将模型实例放置在 Y 轴和 Z 轴上的网格中。然而，由于我们使用的视图矩阵，相机是以 45 度的角度看着这个平面的，所以让我们在 `App::update_uniform_buffer` 中更新视图矩阵，使其直接看向 YZ 平面，以更好地查看我们的模型实例。

```rust,noplaypen
let view = Mat4::look_at_rh(
    point3(6.0, 0.0, 2.0),
    point3(0.0, 0.0, 0.0),
    vec3(0.0, 0.0, 1.0),
);
```

有了一个更好的视角，运行程序并享受它的荣耀吧。

![](../images/4_models.png)

让我们提高一下难度，允许用户决定他们想要渲染多少个模型。在构造函数中为 `App` 结构体添加一个 `models` 字段，并将其初始化为 1。

```rust,noplaypen
struct App {
    // ...
    models: usize,
}
```

将 `App::update_command_buffer` 中的模型索引范围更新为从 0 到 `models` 字段的值。

```rust,noplaypen
let secondary_command_buffers = (0..self.models)
    .map(|i| self.update_secondary_command_buffer(image_index, i))
    .collect::<Result<Vec<_>, _>>()?;
```

现在我们只需要再根据用户输入来增加和减少 `models` 字段的值。首先导入以下 `winit` 类型，我们将需要它们来处理键盘输入。

```rust,noplaypen
use winit::event::{ElementState, VirtualKeyCode};
```

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

运行程序并观察，当你按下左右箭头键时，我们每一帧分配和执行的次级指令缓冲的数量会如何变化。

![](../images/3_models.png)

现在你应该已经熟悉了使用 Vulkan 高效渲染动态帧的基本工具。你可以通过多种方式使用这些工具，每种方式都有不同的性能权衡。未来的教程章节可能会更深入地探讨这个问题，但是使用多个线程并行地记录次级指令缓冲的工作是一种常见的技术，通常可以在现代硬件上获得显著的性能提升。
