# 描述符布局与缓冲

> 原文链接：<https://kylemayes.github.io/vulkanalia/uniform/descriptor_layout_and_buffer.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/21_descriptor_layout.rs) | [shader.vert](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/21/shader.vert) | [shader.frag](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/21/shader.frag)

We're now able to pass arbitrary attributes to the vertex shader for each vertex, but what about global variables? We're going to move on to 3D graphics from this chapter on and that requires a model-view-projection matrix. We could include it as vertex data, but that's a waste of memory and it would require us to update the vertex buffer whenever the transformation changes. The transformation could easily change every single frame.

现在我们可以将每个顶点的任何属性上传到顶点着色器了，但是全局变量怎么办呢？从本章开始我们要走向 3D，这就需要一个模型-试图-投影矩阵（model-view-projection matrix）。我们可以将它包含在顶点数据中，但这是一种浪费内存的做法，而且每当变换发生变化时，我们都需要更新顶点缓冲。而变换很可能每一帧都会发生变化。

The right way to tackle this in Vulkan is to use *resource descriptors*. A descriptor is a way for shaders to freely access resources like buffers and images. We're going to set up a buffer that contains the transformation matrices and have the vertex shader access them through a descriptor. Usage of descriptors consists of three parts:

在 Vulkan 中，正确的解决方式是使用*资源描述符（resource descriptor）*。描述符是一种让着色器自由访问缓冲和图像等资源的方法。我们将设置一个包含变换矩阵的缓冲，并让顶点着色器通过描述符访问它们。描述符的使用包括三个部分：

* Specify a descriptor layout during pipeline creation
* Allocate a descriptor set from a descriptor pool
* Bind the descriptor set during rendering

* 在创建管线时指定描述符布局
* 从描述符池中分配一个描述符集合
* 在渲染时绑定描述符集合

The *descriptor layout* specifies the types of resources that are going to be accessed by the pipeline, just like a render pass specifies the types of attachments that will be accessed. A *descriptor set* specifies the actual buffer or image resources that will be bound to the descriptors, just like a framebuffer specifies the actual image views to bind to render pass attachments. The descriptor set is then bound for the drawing commands just like the vertex buffers and framebuffer.

*描述符布局（descriptor layout）*指定了管线将要访问的资源类型，就像渲染流程指定了将要访问的附件类型一样。*描述符集合（descriptor set）*指定了将要绑定到描述符的实际缓冲或图像资源，就像帧缓冲指定了要绑定到渲染流程附件的实际图像视图一样。然后，描述符集合就像顶点缓冲和帧缓冲一样，绑定到绘制命令中。

There are many types of descriptors, but in this chapter we'll work with uniform buffer objects (UBO). We'll look at other types of descriptors in future chapters, but the basic process is the same. Let's say we have the data we want the vertex shader to have in a struct like this:

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

Then we can copy the data to a `vk::Buffer` and access it through a uniform buffer object descriptor from the vertex shader like this:

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

We're going to update the model, view and projection matrices every frame to make the rectangle from the previous chapter spin around in 3D.

我们将在每一帧中更新模型、视图和投影矩阵，使得上一章中的矩形在 3D 空间中旋转起来。

## 顶点着色器

Modify the vertex shader to include the uniform buffer object like it was specified above. I will assume that you are familiar with MVP transformations. If you're not, see [the resource](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/) mentioned in the first chapter.

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

Note that the order of the `uniform`, `in` and `out` declarations doesn't matter. The `binding` directive is similar to the `location` directive for attributes. We're going to reference this binding in the descriptor layout. The line with `gl_Position` is changed to use the transformations to compute the final position in clip coordinates. Unlike the 2D triangles, the last component of the clip coordinates may not be `1`, which will result in a division when converted to the final normalized device coordinates on the screen. This is used in perspective projection as the *perspective division* and is essential for making closer objects look larger than objects that are further away.

注意 `uniform`、`in` 和 `out` 声明之间的顺序是无关紧要的。`binding` 指令和属性的 `location` 指令类似。我们将会在描述符布局中引用这个绑定。`gl_Position` 所在的那一行被修改为使用变换来计算最终的裁剪坐标。与 2D 三角形不同，裁剪坐标的最后一个分量可能不是 `1`，这将导致在转换为屏幕上的最终归一化设备坐标时进行除法运算。这在透视投影中被称为*透视除法*，对于使得更近的物体看起来比更远的物体更大是至关重要的。

## 描述符集合布局

The next step is to define the UBO on the Rust side and to tell Vulkan about this descriptor in the vertex shader. First we add a few more imports and a type alias:

下一步就是在 Rust 这边定义 UBO，并告诉 Vulkan 在顶点着色器中关于这个描述符的信息。首先我们添加一些导入和类型别名：

```rust,noplaypen
use cgmath::{point3, Deg};

type Mat4 = cgmath::Matrix4<f32>;
```

Then create the `UniformBufferObject` struct:

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

We can exactly match the definition in the shader using data types in the `cgmath` crate. The data in the matrices is binary compatible with the way the shader expects it, so we can later just copy a `UniformBufferObject` to a `vk::Buffer`.

使用 `cgmath` crate 中的数据类型，我们可以完全匹配着色器中的定义。矩阵中的数据与着色器期望的数据是二进制兼容的，所以之后我们可以直接将 `UniformBufferObject` 复制到 `vk::Buffer` 中。

We need to provide details about every descriptor binding used in the shaders for pipeline creation, just like we had to do for every vertex attribute and its `location` index. We'll set up a new function to define all of this information called `^create_descriptor_set_layout`. It should be called right before pipeline creation, because we're going to need it there.

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

Every binding needs to be described through a `vk::DescriptorSetLayoutBinding` struct.

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

The first two fields specify the `binding` used in the shader and the type of descriptor, which is a uniform buffer object. It is possible for the shader variable to represent an array of uniform buffer objects, and `descriptor_count` specifies the number of values in the array. This could be used to specify a transformation for each of the bones in a skeleton for skeletal animation, for example. Our MVP transformation is in a single uniform buffer object, so we're using a `descriptor_count` of `1`.

前两个字段指定了着色器中使用的 `binding` 和描述符的类型 —— 这里是 uniform 缓冲对象。着色器变量可以表示由 uniform 缓冲对象组成的数组，`descriptor_count` 指定了数组中的值的数量。例如，这可以用来为骨骼中的每个骨骼指定一个变换。我们的 MVP 变换在一个单独的 uniform 缓冲对象中，所以我们令 `descriptor_count` 为 `1`。

We also need to specify in which shader stages the descriptor is going to be referenced. The `stage_flags` field can be a combination of `vk::ShaderStageFlags` values or the value `vk::ShaderStageFlags::ALL_GRAPHICS`. In our case, we're only referencing the descriptor from the vertex shader.

我们还需要指定描述符将会在哪些着色器阶段被引用。`stage_flags` 字段可以是一系列 `vk::ShaderStageFlags` 值的组合，也可以是 `vk::ShaderStageFlags::ALL_GRAPHICS`。在我们的场景中，我们只在顶点着色器中使用这个描述符。

There is also an `immutable_samplers` field which is only relevant for image sampling related descriptors, which we'll look at later. You can leave this to its default value.

还有一个 `immutable_samplers` 字段，它只与图像采样相关的描述符有关，我们之后会看到。现在我们可以将它保留为默认值。

All of the descriptor bindings are combined into a single `vk::DescriptorSetLayout` object. Define a new `AppData` field above `pipeline_layout`:

所有描述符绑定会被合并到一个 `vk::DescriptorSetLayout` 对象中。在 `AppData` 中，`pipeline_layout` 的上面新增一个字段：

```rust,noplaypen
struct AppData {
    // ...
    descriptor_set_layout: vk::DescriptorSetLayout,
    pipeline_layout: vk::PipelineLayout,
    // ...
}
```

We can then create it using `create_descriptor_set_layout`. This function accepts a simple `vk::DescriptorSetLayoutCreateInfo` with the array of bindings:

接着我们可以使用 `create_descriptor_set_layout` 来创建它。这个函数接受一个简单的 `vk::DescriptorSetLayoutCreateInfo`，其中包含了绑定的数组：

```rust,noplaypen
let bindings = &[ubo_binding];
let info = vk::DescriptorSetLayoutCreateInfo::builder()
    .bindings(bindings);

data.descriptor_set_layout = device.create_descriptor_set_layout(&info, None)?;
```

We need to specify the descriptor set layout during pipeline creation to tell Vulkan which descriptors the shaders will be using. Descriptor set layouts are specified in the pipeline layout object. Modify the `vk::PipelineLayoutCreateInfo` to reference the layout object:

```rust,noplaypen
let set_layouts = &[data.descriptor_set_layout];
let layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(set_layouts);
```

You may be wondering why it's possible to specify multiple descriptor set layouts here, because a single one already includes all of the bindings. We'll get back to that in the next chapter, where we'll look into descriptor pools and descriptor sets.

The descriptor layout should stick around while we may create new graphics pipelines i.e. until the program ends:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();
    self.device.destroy_descriptor_set_layout(self.data.descriptor_set_layout, None);
    // ...
}
```

## Uniform buffer

In the next chapter we'll specify the buffer that contains the UBO data for the shader, but we need to create this buffer first. We're going to copy new data to the uniform buffer every frame, so it doesn't really make any sense to have a staging buffer. It would just add extra overhead in this case and likely degrade performance instead of improving it.

We should have multiple buffers, because multiple frames may be in flight at the same time and we don't want to update the buffer in preparation of the next frame while a previous one is still reading from it! We could either have a uniform buffer per frame or per swapchain image. However, since we need to refer to the uniform buffer from the command buffer that we have per swapchain image, it makes the most sense to also have a uniform buffer per swapchain image.

To that end, add new `AppData` fields for `uniform_buffers`, and `uniform_buffers_memory`:

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

Similarly, create a new function `create_uniform_buffers` that is called after `create_index_buffer` and allocates the buffers:

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

We're going to write a separate function that updates the uniform buffer with a new transformation every frame, so there will be no `map_memory` here. The uniform data will be used for all draw calls, so the buffer containing it should only be destroyed when we stop rendering. Since it also depends on the number of swapchain images, which could change after a recreation, we'll clean it up in `destroy_swapchain`:

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

This means that we also need to recreate it in `recreate_swapchain`:

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    // ...
    create_framebuffers(&self.device, &mut self.data)?;
    create_uniform_buffers(&self.instance, &self.device, &mut self.data)?;
    create_command_buffers(&self.device, &mut self.data)?;
    Ok(())
}
```

## Updating uniform data

Create a new method `App::update_uniform_buffer` and add a call to it from the `App::render` method right after we wait for the fence for the acquired swapchain image to be signalled:

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

It is important that the uniform buffer is not updated until after this fence is signalled!

As a quick refresher on the usage of fences as introduced in the [`Rendering and Presentation` chapter](../drawing/rendering_and_presentation.html#frames-in-flight), we are using fences so that the GPU can notify the CPU once it is done processing a previously submitted frame. These notifications are used for two purposes: to prevent the CPU from submitting more frames when there are already `MAX_FRAMES_IN_FLIGHT` unfinished frames submitted to the GPU and also to ensure the CPU doesn't alter or delete resources like uniform buffers or command buffers while they are still being used by the GPU to process a frame.

Our uniform buffers are associated with our swapchain images, so we need to be sure that any previous frame that rendered to the acquired swapchain image is complete before we can safely update the uniform buffer. By only updating the uniform buffer after the GPU has notified the CPU that this is the case we can safely do whatever we want with the uniform buffer.

Going back to `App::update_uniform_buffer`, this method will generate a new transformation every frame to make the geometry spin around. We need to add an import to implement this functionality:

```rust,noplaypen
use std::time::Instant;
```

The `Instant` struct will allow us to do precise timekeeping. We'll use this to make sure that the geometry rotates 90 degrees per second regardless of frame rate. Add a field to `App` to track the time the application started and initialize the field to `Instant::now()` in `App::create`:

```rust,noplaypen
struct App {
    // ...
    start: Instant,
}
```

We can now use that field to determine how many seconds it has been since the application started:

```rust,noplaypen
unsafe fn update_uniform_buffer(&self, image_index: usize) -> Result<()> {
    let time = self.start.elapsed().as_secs_f32();

    Ok(())
}
```

We will now define the model, view and projection transformations in the uniform buffer object. The model rotation will be a simple rotation around the Z-axis using the `time` variable:

```rust,noplaypen
let model = Mat4::from_axis_angle(
    vec3(0.0, 0.0, 1.0),
    Deg(90.0) * time
);
```

The `Mat4::from_axis_angle` function creates a transformation matrix from the given rotation angle and rotation axis. Using a rotation angle of `Deg(90.0) * time` accomplishes the purpose of rotating 90 degrees per second.

```rust,noplaypen
let view = Mat4::look_at_rh(
    point3(2.0, 2.0, 2.0),
    point3(0.0, 0.0, 0.0),
    vec3(0.0, 0.0, 1.0),
);
```

For the view transformation I've decided to look at the geometry from above at a 45 degree angle. The `Mat4::look_at_rh` function takes the eye position, center position and up axis as parameters. The `rh` at the end of this function indicates that it uses the "right-handed" coordinate system which is the coordinate system that Vulkan uses.

```rust,noplaypen
let mut proj = cgmath::perspective(
    Deg(45.0),
    self.data.swapchain_extent.width as f32 / self.data.swapchain_extent.height as f32,
    0.1,
    10.0,
);
```

I've chosen to use a perspective projection with a 45 degree vertical field-of-view. The other parameters are the aspect ratio, near and far view planes. It is important to use the current swapchain extent to calculate the aspect ratio to take into account the new width and height of the window after a resize.

```rust,noplaypen
proj[1][1] *= -1.0;
```

`cgmath` was originally designed for OpenGL, where the Y coordinate of the clip coordinates is inverted. The easiest way to compensate for that is to flip the sign on the scaling factor of the Y axis in the projection matrix. If you don't do this, then the image will be rendered upside down.

```rust,noplaypen
let ubo = UniformBufferObject { model, view, proj };
```

Lastly we combine our matrices into a uniform buffer object.

All of the transformations are defined now, so we can copy the data in the uniform buffer object to the current uniform buffer. This happens in exactly the same way as we did for vertex buffers, except without a staging buffer:

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

Using a UBO this way is not the most efficient way to pass frequently changing values to the shader. A more efficient way to pass a small buffer of data to shaders are *push constants*. We may look at these in a future chapter.

If you run the program now, you'll get errors about unbound descriptor sets from the validation layer and nothing will be rendered. In the next chapter we'll look at these descriptor sets, which will actually bind the `vk::Buffer`s to the uniform buffer descriptors so that the shader can access this transformation data and get our program in running order again.
