# 总结

> 原文链接：<https://kylemayes.github.io/vulkanalia/pipeline/conclusion.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/12_graphics_pipeline_complete.rs)

现在我们可以开始组合前几章中创建的所有结构和对象来创建图形管线了！回顾一下我们现在有的对象:

* 着色器阶段 &ndash; 着色器单元定义的图形管线中的可编程阶段
* 固定功能状态 &ndash; 管线中所有定义固定功能阶段的结构，例如输入组装、光栅化器、视口和颜色混合
* 管线布局 &ndash; 定义着色器引用的可以在绘制时更新的 `uniform` 值和推送常量
* 渲染流程 &ndash; 管线阶段中引用的附件，以及它们的用途

这些对象定义了图形管线的方方面面。现在我们可以在 `create_pipeline` 函数的末尾（但要在着色器模块销毁之前）开始填充 `vk::GraphicsPipelineCreateInfo` 了。

```rust,noplaypen
let stages = &[vert_stage, frag_stage];
let info = vk::GraphicsPipelineCreateInfo::builder()
    .stages(stages)
    // continued...
```

我们首先提供一个 `vk::PipelineShaderStageCreateInfo` 结构体的数组。

```rust,noplaypen
    .vertex_input_state(&vertex_input_state)
    .input_assembly_state(&input_assembly_state)
    .viewport_state(&viewport_state)
    .rasterization_state(&rasterization_state)
    .multisample_state(&multisample_state)
    .color_blend_state(&color_blend_state)
```

接着我们引用所有描述固定功能阶段的结构体。

```rust,noplaypen
    .layout(data.pipeline_layout)
```

然后是管线布局。

```rust,noplaypen
    .render_pass(data.render_pass)
    .subpass(0);
```

最后引用之前创建的渲染流程，以及图形管线将要使用的子流程在子流程数组中的索引。在这个渲染管线上使用其他的渲染流程也是可以的，但这些渲染流程之间必须*相互兼容*。[这里](https://www.khronos.org/registry/vulkan/specs/1.2/html/vkspec.html#renderpass-compatibility)给出了关于兼容性的描述，不过本教程中我们不会使用这个特性。

```rust,noplaypen
    .base_pipeline_handle(vk::Pipeline::null()) // 可选.
    .base_pipeline_index(-1)                    // 可选.
```

实际上还有两个参数：`base_pipeline_handle` 和 `base_pipeline_index`。Vulkan 允许你派生一个现有的图形管线来创建新的图形管线。管线派生的意义在于，如果新的管线和旧的管线有很多相似之处，这样做就能减少很多开销；在同一个亲代（parent）派生出的图形管线之间切换也更快。你可以使用 `base_pipeline_handle` 通过句柄来指定一个现有的管线，或者使用 `base_pipeline_index` 通过索引来指定一个即将创建的管线。现在我们只有一个管线，所以我们会简单地指定一个空句柄和一个无效索引。只有在 `vk::GraphicsPipelineCreateInfo` 的 `flags` 字段中也指定了 `vk::PipelineCreateFlags::DERIVATIVE` 标志时，这些值才会被使用。

现在，在 `AppData` 中添加一个字段来存储 `vk::Pipeline` 对象：

```rust,noplaypen
struct AppData {
    // ...
    pipeline: vk::Pipeline,
}
```

然后在 `App::create` 中创建图形管线：

```rust,noplaypen
data.pipeline = device.create_graphics_pipelines(
    vk::PipelineCache::null(), &[info], None)?.0[0];
```

`create_graphics_pipelines` 函数的参数比 Vulkan 中通常的对象创建函数要多。它被设计为可以一次性接受多个 `vk::GraphicsPipelineCreateInfo` 对象并创建多个 `vk::Pipeline` 对象。

第一个参数是一个对 `vk::PipelineCache` 的引用，这个参数是可选的，我们为其传递 `vk::PipelineCache::null()`。管线缓存可以用来在多次调用 `create_graphics_pipelines` 时存储和重用管线创建相关的数据，甚至可以在程序执行结束后从文件中读取缓存。这样可以显著提高管线创建的速度。

图形管线会在所有的绘制操作中使用，所以它也应该在 `App::destroy` 中被销毁：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_pipeline(self.data.pipeline, None);
    // ...
}
```

现在运行程序，来确定*我们一直以来的努力并非全部白费*。现在，我们离看到屏幕上有东西出现不远了。在接下来的几章中，我们将设置交换链图像的实际帧缓冲，并准备绘制指令。
