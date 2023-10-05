# 固定功能

> 原文链接：<https://kylemayes.github.io/vulkanalia/pipeline/fixed_functions.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/10_fixed_functions.rs)

旧的图形学 API 在渲染管线中为大部分阶段提供了默认状态。而在 Vulkan 中，你必须显式指定一切 —— 从视口大小到颜色混合函数。在本章中，我们会创建并填充配置这些固定功能操作所需的所有结构体。

## 顶点输入

`vk::PipelineVertexInputStateCreationInfo` 结构体大致上通过两种方式描述顶点数据的格式：

* 绑定（Bindings） &ndash; 数据之间的间隔，以及数据是逐顶点（per-vertex）还是逐个实例（per-instance）的（参见[实例化](https://en.wikipedia.org/wiki/Geometry_instancing)）
* 属性描述（Attribute descriptions） &ndash; 顶点着色器接收的属性的类型，从哪个绑定中加载数据，以及从哪个偏移量开始加载

因为我们已经在顶点着色器中硬编码了数据，我们会将这个结构体的所有字段都保留默认值，表明我们不需要加载数据。我们会在顶点缓冲章节重新审视这个结构体。

```rust,noplaypen
let vertex_input_state = vk::PipelineVertexInputStateCreateInfo::builder();
```

在 `create_pipeline` 函数中，我们会在 `vk::PipelineShaderStageCreateInfo` 结构体之后添加这个结构体。

<!--
The `vertex_binding_descriptions` and `vertex_attribute_descriptions` fields for this struct that could have been set here would be slices of structs that describe the aforementioned details for loading vertex data.

这一句就不翻译了，免得读者误解。
-->

## 输入装配

`vk::PipelineInputAssemblyStateCreateInfo` 结构体指定了两件事：从顶点中绘制什么样的几何图形，以及是否启用了图元重启（primitive restart）。前者由 `topology` 字段指定，可以有以下值：

* `vk::PrimitiveTopology::POINT_LIST` &ndash; 根据输入顶点绘制点
* `vk::PrimitiveTopology::LINE_LIST` &ndash; 每两个顶点绘制一条线，不重复使用顶点
* `vk::PrimitiveTopology::LINE_STRIP` &ndash; 每两个顶点绘制一条线，每条线的终点是下一条线的起点
* `vk::PrimitiveTopology::TRIANGLE_LIST` &ndash; 每三个顶点绘制一个三角形，不重复使用顶点
* `vk::PrimitiveTopology::TRIANGLE_STRIP` &ndash; 每三个顶点绘制一个三角形，每个三角形的第二个和第三个顶点是下一个三角形的第一和第二个顶点

通常情况下，我们从顶点缓冲中加载的顶点都是按索引顺序排列的，但你也可以使用*元素缓冲（element buffer）*自行指定索引顺序。这就允许你进行一些优化，例如复用顶点。

如果你将 `primitive_restart_enable` 字段设置为 `true`，那么你就可以使用特殊的索引 `0xFFFF` 或 `0xFFFFFFFF` 来打断 `_STRIP` 拓扑模式下的线和三角形。

我们的目的是绘制三角形，所以我们按照以下方式填充结构体：

```rust,noplaypen
let input_assembly_state = vk::PipelineInputAssemblyStateCreateInfo::builder()
    .topology(vk::PrimitiveTopology::TRIANGLE_LIST)
    .primitive_restart_enable(false);
```

## 视口和裁剪

视口用于描述渲染结果将被输出到的帧缓冲域。视口几乎总是从 `(0, 0)` 到 `(width, height)`，在本教程中也是如此。

```rust,noplaypen
let viewport = vk::Viewport::builder()
    .x(0.0)
    .y(0.0)
    .width(data.swapchain_extent.width as f32)
    .height(data.swapchain_extent.height as f32)
    .min_depth(0.0)
    .max_depth(1.0);
```

交换链的尺寸和交换链图像的尺寸并不总是相同。因为我们之后是要将交换链图像用作帧缓冲，所以我们应该使用图像的尺寸。

`min_depth` 和 `max_depth` 值指定了帧缓冲的深度值范围。这些值必须在 `[0.0, 1.0]` 范围内，但 `min_depth` 可以比 `max_depth` 更大。如果你没在整什么骚活，那么你应该使用标准值 `0.0` 和 `1.0`。

视口定义了图像到帧缓冲的映射关系，而裁剪矩形（scissor rectangles）则定义了哪些像素会被实际地存储到帧缓冲中。与其说是变换，裁剪矩形更像是一个过滤器。下图展示了视口和裁剪矩形的区别。注意左边的裁剪矩形只是产生这张图像的可能性之一，只要它比视口大就行。
<!-- 最后一句不是很通顺 -->

![](../images/viewports_scissors.png)

在本教程中，我们只想填充整个帧缓冲域，所以我们会指定一个覆盖整个帧缓冲域的裁剪矩形：

```rust,noplaypen
let scissor = vk::Rect2D::builder()
    .offset(vk::Offset2D { x: 0, y: 0 })
    .extent(data.swapchain_extent);
```

视口和裁剪矩形的信息需要用 `vk::PipelineViewportStateCreateInfo` 结构体来组合在一起。在某些显卡上，你可以使用多个视口和裁剪矩形，所以这个结构体的成员是一个数组。使用多个视口需要启用一个 GPU 特性（参见逻辑设备创建）。

```rust,noplaypen
let viewports = &[viewport];
let scissors = &[scissor];
let viewport_state = vk::PipelineViewportStateCreateInfo::builder()
    .viewports(viewports)
    .scissors(scissors);
```

## 光栅化

光栅化程序将来自顶点着色器的顶点构成的几何图元交给片元着色器。它也负责进行[深度测试](https://en.wikipedia.org/wiki/Z-buffering)、[背面剔除（face culling）](https://en.wikipedia.org/wiki/Back-face_culling)和剪裁测试（scissor test），并且可以配置为输出填充整个多边形的片元还是只输出多边形的边缘（线框渲染）。所有这些都可以通过 `vk::PipelineRasterizationStateCreateInfo` 结构体来配置。

```rust,noplaypen
let rasterization_state = vk::PipelineRasterizationStateCreateInfo::builder()
    .depth_clamp_enable(false)
    // continued...
```

如果 `depth_clamp_enable` 被设为 `true`，在近平面和远平面以外的片元都会被截断到这两个平面上，而不是被丢弃。这在一些特殊情况下很有用，例如生成阴影贴图的时候。使用这个选项需要启用一个 GPU 特性。

```rust,noplaypen
    .rasterizer_discard_enable(false)
```

如果 `rasterizer_discard_enable` 被设为 `true`，那么几何图元就永远不会通过光栅化程序阶段。这基本上就禁用了对帧缓冲的任何输出。

```rust,noplaypen
    .polygon_mode(vk::PolygonMode::FILL)
```

`polygon_mode` 字段决定了几何图元如何生成片元：

* `vk::PolygonMode::FILL` &ndash; 用片元填充多边形的区域
* `vk::PolygonMode::LINE` &ndash; 用线段绘制多边形的边缘
* `vk::PolygonMode::POINT` &ndash; 用点绘制多边形的顶点

使用填充以外的模式都需要启用一个 GPU 特性。

```rust,noplaypen
    .line_width(1.0)
```

`line_width` 成员非常直白，它以图元为单位指定线条的宽度。支持的最大线宽取决于硬件，任何比 `1.0` 更宽的线条都需要启用一个 GPU 特性。

```rust,noplaypen
    .cull_mode(vk::CullModeFlags::BACK)
    .front_face(vk::FrontFace::CLOCKWISE)
```

`cull_mode` 变量决定了使用哪种面剔除。你可以禁用剔除、剔除正面、剔除背面或同时剔除正面和背面。`front_face` 变量用于指定正面的顶点顺序，可以是顺时针或逆时针。

```rust,noplaypen
    .depth_bias_enable(false);
```

光栅化器可以通过添加一个常量值或者根据片元的斜率来改变深度值。这有时用于阴影贴图，但我们不会用到它。只需将 `depth_bias_enable` 设为 `false` 即可。

## 多重采样（multisampling）

`vk::PipelineMultisampleStateCreateInfo` 结构体用于配置多重采样 —— 一种[抗锯齿](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)的方式。它通过组合多个光栅化后的多边形的片元着色器结果来工作。这主要发生在边缘，也是锯齿伪影最明显的地方。因为如果只有一个多边形映射到一个像素，那么它就不需要运行多次片元着色器，所以它比简单地渲染到更高分辨率然后缩小要便宜得多。启用它需要启用一个 GPU 特性。

```rust,noplaypen
let multisample_state = vk::PipelineMultisampleStateCreateInfo::builder()
    .sample_shading_enable(false)
    .rasterization_samples(vk::SampleCountFlags::_1);
```

我们会在后面的章节再讨论多重采样，现在我们先将它禁用。

## 深度测试和模板测试（stencil testing）

如果需要进行深度测试和模板测试，那么除了需要配置深度缓冲和模板缓冲之外，你还需要通过 `vk::PipelineDepthStencilStateCreateInfo` 结构体配置管线。我们现在还不需要这些，所以我们可以忽略它们。我们会在深度缓冲章节再讨论它们。

## 颜色混合

片元着色器返回的颜色需要与帧缓冲中已有的颜色进行混合。混合的方式有两种：

* 混合新值和旧值以产生最终颜色
* 使用位运算组合新值和旧值

有两种结构体可以用于配置颜色混合。`vk::PipelineColorBlendAttachmentState` 可以对每个帧缓冲进行单独的颜色配置，而 `vk::PipelineColorBlendStateCreateInfo` 则可以进行*全局*的颜色混合配置。我们的例子里只有一个帧缓冲：

```rust,noplaypen
let attachment = vk::PipelineColorBlendAttachmentState::builder()
    .color_write_mask(vk::ColorComponentFlags::all())
    .blend_enable(false)
    .src_color_blend_factor(vk::BlendFactor::ONE)  // 可选
    .dst_color_blend_factor(vk::BlendFactor::ZERO) // 可选
    .color_blend_op(vk::BlendOp::ADD)              // 可选
    .src_alpha_blend_factor(vk::BlendFactor::ONE)  // 可选
    .dst_alpha_blend_factor(vk::BlendFactor::ZERO) // 可选
    .alpha_blend_op(vk::BlendOp::ADD);             // 可选
```

我们通过上面这个结构体为帧缓冲配置第一类颜色混合方式，这种方式的运算过程类似于下面的代码：

```rust,noplaypen
if blend_enable {
    final_color.rgb = (src_color_blend_factor * new_color.rgb)
        <color_blend_op> (dst_color_blend_factor * old_color.rgb);
    final_color.a = (src_alpha_blend_factor * new_color.a)
        <alpha_blend_op> (dst_alpha_blend_factor * old_color.a);
} else {
    final_color = new_color;
}

final_color = final_color & color_write_mask;
```

如果 `blend_enable` 被设为 `false`，那么片元着色器返回的新颜色会原封不动地传递到帧缓冲中。否则，这两种混合运算（`color_blend_op` 和 `alpha_blend_op`）会被执行，计算出一个新的颜色。最终的颜色会与 `color_write_mask` 进行位运算，以决定哪些通道会被实际地传递到帧缓冲中。

通常，我们使用颜色混合是为了进行阿尔法合成（alpha blending），基于透明度混合新旧颜色。这种情况下，`final_color` 的计算过程如下：

```c++
final_color.rgb = new_alpha * new_color + (1 - new_alpha) * old_color;
final_color.a = new_alpha.a;
```

这可以通过以下参数实现：

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

你可以在 Vulkan 规范里 `vk::BlendFactor` 和 `vk::BlendOp` 枚举的文档（或者 `vulkanalia` 的文档）中找到所有可能的运算。

第二种结构体（`vk::PipelineColorBlendStateCreateInfo`）引用了所有帧缓冲，并且允许你设置混合常量，你可以在混合运算中使用这些常量。

```rust,noplaypen
let attachments = &[attachment];
let color_blend_state = vk::PipelineColorBlendStateCreateInfo::builder()
    .logic_op_enable(false)
    .logic_op(vk::LogicOp::COPY)
    .attachments(attachments)
    .blend_constants([0.0, 0.0, 0.0, 0.0]);
```

如果你想使用第二种混合方式（位运算结合），你应该把 `logic_op_enable` 设为 `true`。位运算操作可由 `logic_op` 字段指定。注意这会自动禁用第一种混合方式，就好像你把所有帧缓冲的 `blend_enable` 都置为了 `false`！`color_write_mask` 在这种模式下也会被用到，以决定哪些通道会被实际地传递到帧缓冲中。你也可以同时禁用这两种模式，就像我们这里做的一样，这样片元颜色就会原封不动地传递到帧缓冲中。

## 动态状态

有一部分状态是可以在不重新创建管线的情况下改变的。例如视口的大小、线宽和混合常量。如果你想这么做，那么你需要填充一个 `vk::PipelineDynamicStateCreateInfo` 结构体：

```rust,noplaypen
let dynamic_states = &[
    vk::DynamicState::VIEWPORT,
    vk::DynamicState::LINE_WIDTH,
];

let dynamic_state = vk::PipelineDynamicStateCreateInfo::builder()
    .dynamic_states(dynamic_states);
```

这会使得这些值（视口和线宽）的设置被忽略，你需要在绘制时指定这些值。我们会在后面的章节再讨论这个结构体。如果你没有任何动态状态，那么你可以忽略这个结构体。

## 管线布局

你可以在着色器中使用 `uniform` 值，它们类似于动态状态变量，可以在不重新创建着色器的前提下，在绘制时改变着色器的行为。它们通常用于将变换矩阵或是纹理采样器传递给顶点着色器。尽管我们在本章中不会使用它们，但我们仍然需要在管线创建时指定一个空的管线布局。

在 `AppData` 中添加一个 `pipeline_layout` 字段，因为我们会在别的函数中用到它：

```rust,noplaypen
struct AppData {
    // ...
    pipeline_layout: vk::PipelineLayout,
}
```

然后在 `create_pipeline` 函数中，在调用 `destroy_shader_module` 之前创建这个对象：

```rust,noplaypen
let layout_info = vk::PipelineLayoutCreateInfo::builder();

data.pipeline_layout = device.create_pipeline_layout(&layout_info, None)?;
```

这个结构体也用于指定*推式常量* —— 另一种给着色器传递动态值的方式，我们会在后面的章节中介绍。

管线布局会在整个程序的生存期中被引用，所以它应该在 `App::destroy` 中被销毁：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_pipeline_layout(self.data.pipeline_layout, None);
    // ...
}
```

## 结论

这就是所有的固定功能状态！从零开始配置这些状态是一项很大的工作，但好处是我们现在几乎完全了解了图形管线中发生的一切！这降低了因为某些组件的默认状态与你期望的不同，而导致意外行为的可能性。

我们还要创建一样东西来完成图形管线，那就是渲染流程（render pass）。
