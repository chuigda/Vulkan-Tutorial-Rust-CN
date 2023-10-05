# 推送常量

> 原文链接：<https://kylemayes.github.io/vulkanalia/dynamic/push_constants.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

> <span style="display: flex; justify-content: center; margin-bottom: 16px"><img src="../images/i_have_no_idea_what_im_doing.jpg" width="256"></span>前面没有这个声明的章节都是直接从 <https://github.com/Overv/VulkanTutorial> 改编而来。<br/><br/>这一章和后面的章节都是原创，作者并不是 Vulkan 的专家。作者尽力保持了权威的语气，但是这些章节应该被视为一个 Vulkan 初学者的“尽力而为”。<br/><br/>如果你有问题、建议或者修正，请[提交 issue](https://github.com/KyleMayes/vulkanalia/issues)！

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/30_push_constants.rs) | [shader.vert](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/30/shader.vert) | [shader.frag](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/30/shader.frag)

目前为止我们在教程中创建的场景都是静态的。虽然我们可以操作提供模型、视图和投影（MVP）矩阵的 uniform 缓冲来旋转和移动模型，但是我们不能改变*渲染什么*。这是因为程序在初始化的过程中分配和记录指令缓冲的时候就已经决定了要渲染什么。

在接下来的几章中，我们将会探索多种用于渲染动态场景的技巧。我们首先来看一下*推送常量*，这是 Vulkan 的一个特性，它允许我们轻松高效地“推送”动态数据到着色器。仅靠推送常量本身并不能实现动态场景，但是在接下来的几章中，你会逐渐明白它的用处。

## 推式常量与 uniform 缓冲

之前我们已经在使用另一种能将动态数据提供给顶点着色器的 Vulkan 特性了：那就是 uniform 缓冲。在渲染每一帧时，`App::update_uniform_buffer` 方法都会根据模型当前的旋转角度计算出新的 MVP 矩阵，并将这些矩阵复制到 uniform 缓冲中。然后顶点着色器从 uniform 缓冲中读取这些矩阵，以确定模型的顶点在屏幕上的位置。

目前为止这种方法工作得很好，那么为什么我们要用推送常量取而代之呢？相比于 uniform 缓冲，推送常量的优势之一是速度：更新推送常量比复制新数据到 uniform 缓冲要快得多。如果有大量需要频繁更新的值，这种差异会很快累积起来。

当然，这里有一个限制：使用推送常量提供给着色器的数据量有一个*非常*有限的最大值。这个最大值因设备而异，由 `vk::PhysicalDeviceLimits` 的 `max_push_constants_size` 字段以字节为单位指定。Vulkan 要求这个限制至少为[128字节](https://www.khronos.org/registry/vulkan/specs/1.2/html/chap33.html#limits-minmax)（见表 32），但实际设备上的最大值也不会比 128 高多少。即使是 RTX 3080 这样的高端硬件，这个最大值也只有 256 字节。

如果我们想用推送常量来向矩阵提供我们的 MVP 矩阵，那么我们马上就会撞到这个限制。MVP 矩阵太大了，无法被可靠地放入推送常量中：每个矩阵是 64 字节（16 × 4 字节浮点数），总共 192 字节。当然，我们可以写两套代码，一套用于处理推送常量 >= 192 字节的设备，另一套用于处理无法满足这个需求的设备，但是我们还有更简单的方法。

一种方法就是将 MVP 矩阵预先相乘，获得一个单独的矩阵。另一种方法是只将模型矩阵作为推送常量提供，而将视图和投影矩阵留在 uniform 缓冲中。这两种方法都可以为我们提供至少 64 字节的余量。在本章中，我们将采用第二种方法来探索推送常量。

为什么只将推送常量用于模型矩阵呢？在 `App::update_uniform_buffer` 方法中，你可以注意到 `model` 矩阵随着 `time` 的增加，在每一帧中都会有所改变，而 `view` 矩阵是静态的，`proj` 矩阵则只会在窗口大小发生变化的时候改变。这就使得我们可以只在窗口大小发生变化时更新包含视图和投影矩阵的 uniform 缓冲，而将推送常量用于不断变化的模型矩阵。

当然，在实际的应用程序中，视图矩阵不太可能是静态的。例如，如果你在构建一个第一人称游戏，`view` 矩阵可能会随着玩家在游戏世界中移动而快速变化。然而，即使视图和投影矩阵每一帧都发生改变，至少它们也会在你渲染的所有模型之间共享。也就是说你可以在每一帧中更新 uniform 缓冲，以提供共享的视图和投影矩阵，并使用推送常量来为场景中每个模型提供模型矩阵。

## 推送模型矩阵

文字部分就到这里，现在让我们开始吧，将顶点着色器中的模型矩阵从 uniform 缓冲对象移动到推送常量中。别忘了重新编译顶点着色器！

```glsl
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 view;
    mat4 proj;
} ubo;

layout(push_constant) uniform PushConstants {
    mat4 model;
} pcs;

// ...

void main() {
    gl_Position = ubo.proj * ubo.view * pcs.model * vec4(inPosition, 1.0);
    // ...
}
```

注意不同于 uniform 缓冲对象，推送常量的布局是 `push_constant`，而不是像 `push_constant = 0` 这样的东西。这是因为我们在调用图形管线时只能提供一个推送常量集合，并且如上所述，这个集合非常有限。

从 `UniformBufferObject` 结构体中移除 `model`，因为从现在开始我们会为模型矩阵使用推送常量。

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    view: Mat4,
    proj: Mat4,
}
```

Also remove `model` from the `App::update_command_buffers` method.

同时从 `App::update_command_buffers` 方法中移除 `model`。

```rust,noplaypen
let view = Mat4::look_at_rh(
    point3(2.0, 2.0, 2.0),
    point3(0.0, 0.0, 0.0),
    vec3(0.0, 0.0, 1.0),
);

let correction = Mat4::new(
    1.0,  0.0,       0.0, 0.0,
    0.0, -1.0,       0.0, 0.0,
    0.0,  0.0, 1.0 / 2.0, 0.0,
    0.0,  0.0, 1.0 / 2.0, .0,
);

let proj = correction * cgmath::perspective(
    Deg(45.0),
    self.data.swapchain_extent.width as f32 / self.data.swapchain_extent.height as f32,
    0.1,
    10.0,
);

let ubo = UniformBufferObject { view, proj };

// ...
```

我们需要更新图形管线的布局，将我们的新推送常量告知 Vulkan。在 `create_pipeline` 函数中，你可以看到我们之前已经将描述符集合布局提供给了 `create_pipeline_layout`。描述符集合布局描述了我们着色器中使用的 uniform 缓冲对象和纹理采样器，而现在我们需要使用推送常量范围 `vk::PushConstantRange` 来描述图形管线中着色器将要访问的推送常量。

```rust,noplaypen
let vert_push_constant_range = vk::PushConstantRange::builder()
    .stage_flags(vk::ShaderStageFlags::VERTEX)
    .offset(0)
    .size(64 /* 16 × 4 byte floats */);

let set_layouts = &[data.descriptor_set_layout];
let push_constant_ranges = &[vert_push_constant_range];
let layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(set_layouts)
    .push_constant_ranges(push_constant_ranges);

data.pipeline_layout = device.create_pipeline_layout(&layout_info, None)?;
```

这里的推送常量范围说明了顶点着色器访问的推送常量可以在提供给图形管线的推送常量的开头找到，大小相当于一个 `mat4`。

有了这些，我们就可以开始将模型矩阵推送到顶点着色器了。推送常量会被直接记录到提交给 GPU 的指令缓冲中，这就是为什么它们如此快速，这也解释了为什么它们的大小如此有限。

在 `create_command_buffers` 函数中，定义一个模型矩阵，并在我们记录绘制指令之前，使用 `cmd_push_constants` 将其作为推送常量添加到指令缓冲中。

```rust,noplaypen
let model = Mat4::from_axis_angle(
    vec3(0.0, 0.0, 1.0),
    Deg(0.0)
);

let model_bytes = &*slice_from_raw_parts(
    &model as *const Mat4 as *const u8,
    size_of::<Mat4>()
);

for (i, command_buffer) in data.command_buffers.iter().enumerate() {
    // ...

    device.cmd_push_constants(
        *command_buffer,
        data.pipeline_layout,
        vk::ShaderStageFlags::VERTEX,
        0,
        model_bytes,
    );
    device.cmd_draw_indexed(*command_buffer, data.indices.len() as u32, 1, 0, 0, 0);

    // ...
}
```

如果你现在运行程序，你会看到熟悉的模型，但是它不再旋转了！我们不再在每一帧中更新 uniform 缓冲对象中的模型矩阵，而是将其编码到指令缓冲中，正如之前所讨论的，指令缓冲永远不会被更新。这进一步凸显了以某种方式更新指令缓冲的需求，这个话题将在下一章中讨论。现在，让我们在片元着色器中再添加一个推送常量，然后结束本章。

## 推送透明度

接下来我们将在片元着色器中添加一个推送常量，我们可以用它来控制模型的不透明度。首先修改片元着色器，将推送常量包含进去，并将其用作片元颜色的 alpha 通道。同样，一定要重新编译着色器！

```glsl
#version 450

layout(binding = 1) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    layout(offset = 64) float opacity;
} pcs;

// ...

void main() {
    outColor = vec4(texture(texSampler, fragTexCoord).rgb, pcs.opacity);
}
```

这次我们为推送常量值指定了一个偏移量。记住，推送常量在图形管线中的所有着色器之间是共享的，所以我们需要考虑到推送常量的前 64 字节已经被顶点着色器中使用的模型矩阵占用了。

在管线布局中为新的不透明度推送常量添加一个推送常量范围。

```rust,noplaypen
let vert_push_constant_range = vk::PushConstantRange::builder()
    .stage_flags(vk::ShaderStageFlags::VERTEX)
    .offset(0)
    .size(64 /* 16 × 4 byte floats */);

let frag_push_constant_range = vk::PushConstantRange::builder()
    .stage_flags(vk::ShaderStageFlags::FRAGMENT)
    .offset(64)
    .size(4);

let set_layouts = &[data.descriptor_set_layout];
let push_constant_ranges = &[vert_push_constant_range, frag_push_constant_range];
let layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(set_layouts)
    .push_constant_ranges(push_constant_ranges);

data.pipeline_layout = device.create_pipeline_layout(&layout_info, None)?;
```

最后，在 `create_command_buffers` 中，在为模型矩阵调用 `cmd_push_constants` 之后，再添加一个 `cmd_push_constants` 的调用。

```rust,noplaypen
device.cmd_push_constants(
    *command_buffer,
    data.pipeline_layout,
    vk::ShaderStageFlags::VERTEX,
    0,
    model_bytes,
);
device.cmd_push_constants(
    *command_buffer,
    data.pipeline_layout,
    vk::ShaderStageFlags::FRAGMENT,
    64,
    &0.25f32.to_ne_bytes()[..],
);
device.cmd_draw_indexed(*command_buffer, data.indices.len() as u32, 1, 0, 0, 0);
```

这里我们通过在模型矩阵的 64 字节之后将不透明度 `0.25` 记录到指令缓冲中，来向片元着色器提供不透明度。然而，如果你现在运行程序，你会发现模型仍然完全不透明！

回看[固定功能](../pipeline/fixed_functions.html#color-blending)那一章，我们讨论过如果要渲染透明几何体，就需要设置阿尔法合成。然而，那时我们禁用了阿尔法合成。所以现在我们要在 `create_pipeline` 函数中，更新 `vk::PipelineColorBlendAttachmentState`，以启用阿尔法合成，就像那一章中描述的那样。

```rust,noplaypen
let attachment = vk::PipelineColorBlendAttachmentState::builder()
    .color_write_mask(vk::ColorComponentFlags::all())
    .blend_enable(true)
    .src_color_blend_factor(vk::BlendFactor::SRC_ALPHA)
    .dst_color_blend_factor(vk::BlendFactor::ONE_MINUS_SRC_ALPHA)
    .color_blend_op(vk::BlendOp::ADD)
    .src_alpha_blend_factor(vk::BlendFactor::ONE)
    .dst_alpha_blend_factor(vk::BlendFactor::ZERO)
    .alpha_blend_op(vk::BlendOp::ADD);
```

运行程序，看看我们现在幽灵一般的模型。

![](../images/opacity_push_constant.png)

成功了！
