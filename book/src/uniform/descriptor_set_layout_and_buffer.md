# 描述符集合布局与缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/uniform/descriptor_set_layout_and_buffer.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/21_descriptor_set_layout.rs) | [shader.vert](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/21/shader.vert) | [shader.frag](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/21/shader.frag)

现在我们可以将任何顶点属性上传到顶点着色器了，但是全局变量怎么办呢？从本章开始我们要走向 3D，这就需要一个模型-视图-投影矩阵（model-view-projection matrix）。我们可以将它包含在顶点数据中，但这是一种浪费内存的做法，而且每当变换发生变化时，我们都需要更新顶点缓冲。而变换很可能每一帧都会发生变化。

在 Vulkan 中，正确的解决方式是使用*资源描述符（resource descriptor）*。描述符是一种能够让着色器自由访问缓冲和图像等资源的方法。我们将设置一个包含变换矩阵的缓冲，并让顶点着色器通过描述符访问它们。描述符的使用包括三个部分：

* 在创建管线时指定描述符集合布局
* 从描述符池中分配一个描述符集合
* 在渲染时绑定描述符集合

*描述符集合布局（descriptor set layout）*指定了管线将要访问的资源类型，就像渲染流程指定了将要访问的附件类型一样。*描述符集合（descriptor set）*指定了将要绑定到描述符的实际缓冲或图像资源，就像帧缓冲指定了要绑定到渲染流程附件的实际图像视图一样。然后，描述符集合就像顶点缓冲和帧缓冲一样，绑定到绘制指令中。

描述符有很多种，但在本章中我们将使用 uniform 缓冲对象（uniform buffer object，UBO）。我们将在以后的章节中介绍其他类型的描述符，但基本过程是相同的。假设我们有一个结构体，其中包含了我们希望顶点着色器能够访问的数据：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    model: Mat4,
    view: Mat4,
    proj: Mat4,
}
```

接着我们将数据复制到一个 `vk::Buffer` 中，然后在顶点着色器中像这样通过一个 uniform 缓冲对象描述符来访问它：

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

// ...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

我们将在每一帧中更新模型、视图和投影矩阵，使得上一章中的矩形在 3D 空间中旋转起来。

## 顶点着色器

修改顶点着色器，像上面说的那样将 uniform 缓冲对象包含进来。我假设你已经熟悉了 MVP 变换。如果你不熟悉，可以参考第一章中提到的[资源](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)。

```glsl
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

注意 `uniform`、`in` 和 `out` 声明之间的顺序是无关紧要的。`binding` 指令和属性的 `location` 指令类似。我们将会在描述符集合布局中引用这个绑定。`gl_Position` 所在的那一行被修改为使用变换来计算最终的裁剪坐标。与 2D 三角形不同，裁剪坐标的最后一个分量可能不是 `1`，这将导致在转换为屏幕上的最终归一化设备坐标时进行除法运算。这在透视投影中被称为*透视除法*，这对于实现今大远小的视觉效果是至关重要的。

## 描述符集合布局（descriptor set layout）

下一步就是在 Rust 这边定义 UBO 中的数据，并告诉 Vulkan 在顶点着色器中关于这个描述符的信息。首先我们添加一些导入和类型别名：

```rust,noplaypen
use cgmath::{point3, Deg};

type Mat4 = cgmath::Matrix4<f32>;
```

接着创建 `UniformBufferObject` 结构体：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    model: Mat4,
    view: Mat4,
    proj: Mat4,
}
```

使用 `cgmath` crate 中的数据类型，我们可以完全匹配着色器中的定义。矩阵中的数据与着色器期望的数据是二进制兼容的，所以之后我们可以直接将 `UniformBufferObject` 复制到 `vk::Buffer` 中。

我们需要提供管线创建时着色器中使用的每个描述符绑定的详细信息，就像我们为每个顶点属性及其 `location` 索引所做的那样。我们将创建一个名为 `create_descriptor_set_layout` 的新的函数来定义所有这些信息。它应该在创建管线之前被调用，因为我们将在创建管线时用到这些信息。

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_descriptor_set_layout(&device, &mut data)?;
        create_pipeline(&device, &mut data)?;
        // ...
    }
}


unsafe fn create_descriptor_set_layout(
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    Ok(())
}
```

每个绑定都需要使用 `vk::DescriptorSetLayoutBinding` 来描述。

```rust,noplaypen
unsafe fn create_descriptor_set_layout(
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    let ubo_binding = vk::DescriptorSetLayoutBinding::builder()
        .binding(0)
        .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
        .descriptor_count(1)
        .stage_flags(vk::ShaderStageFlags::VERTEX);

    Ok(())
}
```
前两个字段指定了着色器中使用的 `binding` 和描述符的类型 —— 这里是 uniform 缓冲对象。着色器变量可以表示由 uniform 缓冲对象组成的数组，`descriptor_count` 指定了数组中的值的数量。在实际的应用中，uniform 缓冲对象数组大有用处，例如它可以用来为骨骼中的每个骨骼指定一个变换。我们的 MVP 变换只用到了一个 uniform 缓冲对象，所以我们将 `descriptor_count` 设为 `1`。

我们还需要指定描述符将会在哪些着色器阶段被引用。`stage_flags` 字段可以是一系列 `vk::ShaderStageFlags` 值的组合，也可以是 `vk::ShaderStageFlags::ALL_GRAPHICS`。在我们的场景中，我们只在顶点着色器中使用这个描述符。

还有一个 `immutable_samplers` 字段，它只与图像采样相关的描述符有关，我们之后会看到。现在我们可以将它保留为默认值。

所有描述符绑定会被合并到一个 `vk::DescriptorSetLayout` 对象中。在 `AppData` 中，`pipeline_layout` 的上面新增一个字段：

```rust,noplaypen
struct AppData {
    // ...
    descriptor_set_layout: vk::DescriptorSetLayout,
    pipeline_layout: vk::PipelineLayout,
    // ...
}
```

接着我们可以使用 `create_descriptor_set_layout` 来创建它。这个函数接受一个简单的 `vk::DescriptorSetLayoutCreateInfo`，其中包含了绑定的数组：

```rust,noplaypen
let bindings = &[ubo_binding];
let info = vk::DescriptorSetLayoutCreateInfo::builder()
    .bindings(bindings);

data.descriptor_set_layout = device.create_descriptor_set_layout(&info, None)?;
```

我们需要在创建管线时指定描述符集合布局，以告诉 Vulkan 着色器将会使用哪些描述符。修改 `vk::PipelineLayoutCreateInfo` 来引用描述符集合布局对象：

```rust,noplaypen

```rust,noplaypen
let set_layouts = &[data.descriptor_set_layout];
let layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(set_layouts);
```

你可能会好奇，为什么一个描述符集合布局就可以囊括所有绑定，这里却可以指定多个描述符集合布局。我们会在下一章讨论描述符池和描述符集合时再来审视这个问题。

描述符集合布局应该在我们创建新的图形管线时一直存在，直到程序结束：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_descriptor_set_layout(self.data.descriptor_set_layout, None);
    // ...
}
```

## Uniform 缓冲

在下一章中我们将会为 shader 指定包含了 UBO 数据的缓冲，但我们得先创建这个缓冲。我们将在每一帧中都将新的数据复制到 uniform 缓冲中，所以使用一个暂存缓冲并没有什么意义。在这种情况下，暂存缓冲对提升性能没有帮助，只会增加额外的开销，反而可能降低性能。

我们应该有多个缓冲，因为可能有多个帧同时在参与渲染，并且我们并不希望在前一帧仍在读取缓冲时就为了下一帧更新缓冲！我们可以为每一帧或每一张交换链图像都创建一个 uniform 缓冲。然而，由于我们需要在每一张交换链图像的指令缓冲中引用 uniform 缓冲，所以为每一张交换链图像创建一个 uniform 缓冲是最合理的。

为此，在 `AppData` 中新增 `uniform_buffers` 和 `uniform_buffers_memory` 字段：

```rust,noplaypen
struct AppData {
    // ...
    index_buffer: vk::Buffer,
    index_buffer_memory: vk::DeviceMemory,
    uniform_buffers: Vec<vk::Buffer>,
    uniform_buffers_memory: Vec<vk::DeviceMemory>,
    // ...
}
```

类似地，创建一个新函数 `create_uniform_buffers`，并在 `create_index_buffer` 之后调用它来分配缓冲：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_vertex_buffer(&instance, &device, &mut data)?;
        create_index_buffer(&instance, &device, &mut data)?;
        create_uniform_buffers(&instance, &device, &mut data)?;
        // ...
    }
}

unsafe fn create_uniform_buffers(
    instance: &Instance,
    device: &Device,
    data: &mut AppData,
) -> Result<()> {
    data.uniform_buffers.clear();
    data.uniform_buffers_memory.clear();

    for _ in 0..data.swapchain_images.len() {
        let (uniform_buffer, uniform_buffer_memory) = create_buffer(
            instance,
            device,
            data,
            size_of::<UniformBufferObject>() as u64,
            vk::BufferUsageFlags::UNIFORM_BUFFER,
            vk::MemoryPropertyFlags::HOST_COHERENT | vk::MemoryPropertyFlags::HOST_VISIBLE,
        )?;

        data.uniform_buffers.push(uniform_buffer);
        data.uniform_buffers_memory.push(uniform_buffer_memory);
    }

    Ok(())
}
```

我们会另写一个函数来在每一帧中使用新的变换更新 uniform 缓冲，所以这里不需要 `map_memory`。uniform 数据将用于所有绘制调用，所以包含它的缓冲只应该在我们停止渲染时才被销毁。由于它还取决于交换链图像的数量，而交换链图像的数量在重建后可能会发生变化，所以我们将在 `destroy_swapchain` 中清理它：

```rust,noplaypen
unsafe fn destroy_swapchain(&mut self) {
    self.data.uniform_buffers
        .iter()
        .for_each(|b| self.device.destroy_buffer(*b, None));
    self.data.uniform_buffers_memory
        .iter()
        .for_each(|m| self.device.free_memory(*m, None));
    // ...
}
```

这也就意味着我们也得在 `recreate_swapchain` 中重建它：

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    // ...
    create_framebuffers(&self.device, &mut self.data)?;
    create_uniform_buffers(&self.instance, &self.device, &mut self.data)?;
    create_command_buffers(&self.device, &mut self.data)?;
    Ok(())
}
```

## 更新 uniform 数据

创建一个新的方法 `App::update_uniform_buffer` 并在 `App::render` 方法中获取到交换链图像之后调用它：

```rust,noplaypen
impl App {
    unsafe fn render(&mut self, window: &Window) -> Result<()> {
        // ...

        if !self.data.images_in_flight[image_index as usize].is_null() {
            self.device.wait_for_fences(
                &[self.data.images_in_flight[image_index as usize]],
                true,
                u64::max_value(),
            )?;
        }

        self.data.images_in_flight[image_index as usize] =
            self.data.in_flight_fences[self.frame];

        self.update_uniform_buffer(image_index)?;

        // ...
    }

    unsafe fn update_uniform_buffer(&self, image_index: usize) -> Result<()> {
        Ok(())
    }
}
```

注意在此栅栏 (fence) 发出信号之前不要更新 uniform 缓冲！

快速回顾一下在 [`渲染与呈现` 章节](../drawing/rendering_and_presentation.html#frames-in-flight)中介绍过的栅栏的用法，我们使用栅栏 (fences) 来让 GPU 在处理完之前提交的帧后通知 CPU。这些通知有两个用途：防止 CPU 在已经提交了 `MAX_FRAMES_IN_FLIGHT` 个未完成的帧给 GPU 时继续提交更多的帧；确保 CPU 不会在 GPU 仍在使用资源（如 uniform 缓冲或指令缓冲）处理帧时修改或删除这些资源。

我们的 uniform 缓冲与交换链图像相关联，所以在获取到交换链图像之后，我们需要确保任何之前渲染到这一张图像的帧都已经完成，然后我们才能安全地更新 uniform 缓冲。只有在 GPU 通知 CPU 这种情况发生后才更新 uniform 缓冲，我们才能安全地对 uniform 缓冲做任何操作。

回到 `App::update_uniform_buffer`，这个方法将会在每一帧中生成一个新的变换，使得几何体旋转起来。我们需要添加一个导入来实现这个功能：

```rust,noplaypen
use std::time::Instant;
```

`Instant` 结构体提供了精准的时间记录功能。我们将使用它来确保几何体每秒旋转 90 度，而不管帧率如何。在 `App` 中添加一个字段来跟踪应用程序启动的时间，并在 `App::create` 中将该字段初始化为 `Instant::now()`：

```rust,noplaypen
struct App {
    // ...
    start: Instant,
}
```

现在我们可以使用该字段来确定应用程序启动后经过了多少秒：

```rust,noplaypen
unsafe fn update_uniform_buffer(&self, image_index: usize) -> Result<()> {
    let time = self.start.elapsed().as_secs_f32();

    Ok(())
}
```

现在我们定义 uniform 缓冲对象中的模型、视图和投影变换。模型旋转将是一个简单的绕 Z 轴旋转，使用 `time` 变量：

```rust,noplaypen
let model = Mat4::from_axis_angle(
    vec3(0.0, 0.0, 1.0),
    Deg(90.0) * time
);
```

`Mat4::from_axis_angle` 函数根据指定的旋转角度和转轴创建一个变换矩阵。使用 `Deg(90.0) * time` 作为旋转角度可以实现每秒旋转 90 度的目的。

```rust,noplaypen
let view = Mat4::look_at_rh(
    point3(2.0, 2.0, 2.0),
    point3(0.0, 0.0, 0.0),
    vec3(0.0, 0.0, 1.0),
);
```

至于视图变换，我决定从上方以 45 度的角度看几何体。`Mat4::look_at_rh` 函数接受眼睛位置、中心位置和上轴（up axis）作为参数。这个函数名中的 `rh` 表示它使用的是“右手系”，这也是 Vulkan 使用的坐标系。

```rust,noplaypen
let mut proj = cgmath::perspective(
    Deg(45.0),
    self.data.swapchain_extent.width as f32 / self.data.swapchain_extent.height as f32,
    0.1,
    10.0,
);
```

我决定使用一个在垂直方向上具有 45 度视野（field-of-view）的透视投影。其他的参数分别是宽高比、近视平面和远视平面。使用当前的交换链的交换范围来计算宽高比是很重要的，这样可以将窗口在调整大小后的新尺寸考虑在内。

```rust,noplaypen
proj[1][1] *= -1.0;
```

`cgmath` 原先是为 OpenGL 设计的，因此裁剪空间坐标中的 Y 坐标是反的。要修复这一点，最简单的方法是在反转投影矩阵的 Y 轴缩放因子的符号。如果不这样做，渲染出来的图像会呈现为上下翻转。

```rust,noplaypen
let ubo = UniformBufferObject { model, view, proj };
```

最后我们将矩阵组合成一个 uniform 缓冲对象。

现在我们已经定义了所有的变换，可以将 uniform 缓冲对象中的数据复制到当前的 uniform 缓冲中了。这与我们为顶点缓冲所做的完全相同，除了没有用到暂存缓冲：

```rust,noplaypen
let memory = self.device.map_memory(
    self.data.uniform_buffers_memory[image_index],
    0,
    size_of::<UniformBufferObject>() as u64,
    vk::MemoryMapFlags::empty(),
)?;

memcpy(&ubo, memory.cast(), 1);

self.device.unmap_memory(self.data.uniform_buffers_memory[image_index]);
```

以这种方式使用 UBO 并不是将频繁变化的值传递给着色器的最有效的方法。要将少量数据传递给着色器，更有效的方法是使用*推送常量（push constants）*。我们会在以后的章节中介绍它们。

如果你现在运行程序，你将会得到来自校验层的未绑定描述符集合的错误，并且什么都不会被渲染。在下一章中我们将会看到这些描述符集合，它们将会将 `vk::Buffer` 绑定到 uniform 缓冲描述符，这样着色器就可以访问这些变换数据，然后我们的程序就可以正常运行了。
