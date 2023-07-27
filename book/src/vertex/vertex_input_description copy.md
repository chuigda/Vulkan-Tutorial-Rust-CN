# Vertex input description
# 顶点输入描述
> 原文链接：<https://kylemayes.github.io/vulkanalia/vertex/vertex_input_description.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码：** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/17_vertex_input.rs) | [shader.vert](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/shaders/17/shader.vert) | [shader.frag](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/shaders/17/shader.frag)

In the next few chapters, we're going to replace the hardcoded vertex data in the vertex shader with a vertex buffer in memory. We'll start with the easiest approach of creating a CPU visible buffer copying the vertex data into it directly, and after that we'll see how to use a staging buffer to copy the vertex data to high performance memory.

在接下来的几章中，我们将用内存中的顶点缓冲替换顶点着色器中的硬编码顶点数据。我们将从最简单的方法开始，即创建一个CPU可见的缓冲区，并直接将顶点数据复制到其中。之后，我们将学习如何使用临时缓冲区将顶点数据复制到高性能内存中。

## 顶点着色器

First change the vertex shader to no longer include the vertex data in the shader code itself. The vertex shader takes input from a vertex buffer using the `in` keyword.

首先修改顶点着色器，不再在着色器代码本身中包含顶点数据。顶点着色器将使用 `in` 关键字从顶点缓冲区中获取输入。

```glsl
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

The `inPosition` and `inColor` variables are *vertex attributes*. They're properties that are specified per-vertex in the vertex buffer, just like we manually specified a position and color per vertex using the two arrays. Make sure to recompile the vertex shader!

`inPosition` 和 `inColor` 变量是*顶点属性*。它们是在顶点缓冲区中为每个顶点指定的属性，就像我们之前手动使用两个数组为每个顶点指定了位置和颜色一样。确保重新编译顶点着色器！

Just like `fragColor`, the `layout(location = x)` annotations assign indices to the inputs that we can later use to reference them. It is important to know that some types, like `dvec3` 64 bit vectors, use multiple *slots*. That means that the index after it must be at least 2 higher:

与 fragColor 类似，layout(location = x) 注解为输入变量分配了索引，我们稍后可以用它们来引用这些变量。重要的是要知道，某些类型（例如 dvec3 64位向量）使用多个槽位。这意味着在它之后的索引必须至少比前一个高2：

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

You can find more info about the layout qualifier in the [OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)).

## Vertex data

We're moving the vertex data from the shader code to an array in the code of our program. We'll start by adding a few more imports to our program.

```rust,noplaypen
use std::mem::size_of;

use nalgebra_glm as glm;
use lazy_static::lazy_static;
```

`size_of` will be used to calculate the size of the vertex data we'll be defining while `nalgebra-glm` defines the vector types we need. The `lazy_static!` macro will be used to define the vertex data since the vector constructors in `nalgebra-glm` are not yet `const fn`s.

Next, create a new `#[repr(C)]` structure called `Vertex` with the two attributes that we're going to use in the vertex shader inside it and add a simple constructor:

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Vertex {
    pos: glm::Vec2,
    color: glm::Vec3,
}

impl Vertex {
    fn new(pos: glm::Vec2, color: glm::Vec3) -> Self {
        Self { pos, color }
    }
}
```

`nalgebra-glm` conveniently provides us with Rust types that exactly match the vector types used in the shader language.

```rust,noplaypen
lazy_static! {
    static ref VERTICES: Vec<Vertex> = vec![
        Vertex::new(glm::vec2(0.0, -0.5), glm::vec3(1.0, 0.0, 0.0)),
        Vertex::new(glm::vec2(0.5, 0.5), glm::vec3(0.0, 1.0, 0.0)),
        Vertex::new(glm::vec2(-0.5, 0.5), glm::vec3(0.0, 0.0, 1.0)),
    ];
}
```

Now use the `Vertex` structure to specify a list of vertex data. We're using exactly the same position and color values as before, but now they're combined into one array of vertices. This is known as *interleaving* vertex attributes.

## Binding descriptions

The next step is to tell Vulkan how to pass this data format to the vertex shader once it's been uploaded into GPU memory. There are two types of structures needed to convey this information.

The first structure is `vk::VertexInputBindingDescription` and we'll add a method to the `Vertex` struct to populate it with the right data.

```rust,noplaypen
impl Vertex {
    fn binding_description() -> vk::VertexInputBindingDescription {
    }
}
```

A vertex binding describes at which rate to load data from memory throughout the vertices. It specifies the number of bytes between data entries and whether to move to the next data entry after each vertex or after each instance.

```rust,noplaypen
vk::VertexInputBindingDescription::builder()
    .binding(0)
    .stride(size_of::<Vertex>() as u32)
    .input_rate(vk::VertexInputRate::VERTEX)
    .build()
```

All of our per-vertex data is packed together in one array, so we're only going to have one binding. The `binding` parameter specifies the index of the binding in the array of bindings. The `stride` parameter specifies the number of bytes from one entry to the next, and the `input_rate` parameter can have one of the following values:

* `vk::VertexInputRate::VERTEX` &ndash; Move to the next data entry after each vertex
* `vk::VertexInputRate::INSTANCE` &ndash; Move to the next data entry after each instance

We're not going to use instanced rendering, so we'll stick to per-vertex data.

## Attribute descriptions

The second structure that describes how to handle vertex input is `vk::VertexInputAttributeDescription`. We're going to add another helper method to `Vertex` to fill in these structs.

```rust,noplaypen
impl Vertex {
    fn attribute_descriptions() -> [vk::VertexInputAttributeDescription; 2] {
    }
}
```

As the function prototype indicates, there are going to be two of these structures. An attribute description struct describes how to extract a vertex attribute from a chunk of vertex data originating from a binding description. We have two attributes, position and color, so we need two attribute description structs.

```rust,noplaypen
let pos = vk::VertexInputAttributeDescription::builder()
    .binding(0)
    .location(0)
    .format(vk::Format::R32G32_SFLOAT)
    .offset(0)
    .build();
```

The `binding` parameter tells Vulkan from which binding the per-vertex data comes. The `location` parameter references the `location` directive of the input in the vertex shader. The input in the vertex shader with location `0` is the position, which has two 32-bit float components.

The `format` parameter describes the type of data for the attribute. A bit confusingly, the formats are specified using the same enumeration as color formats. The following shader types and formats are commonly used together:

* `f32` &ndash; `vk::Format::R32_SFLOAT`&nbsp;
* `glm::Vec2` &ndash; `vk::Format::R32G32_SFLOAT`&nbsp;
* `glm::Vec3` &ndash; `vk::Format::R32G32B32_SFLOAT`&nbsp;
* `glm::Vec4` &ndash; `vk::Format::R32G32B32A32_SFLOAT`&nbsp;

As you can see, you should use the format where the amount of color channels matches the number of components in the shader data type. It is allowed to use more channels than the number of components in the shader, but they will be silently discarded. If the number of channels is lower than the number of components, then the BGA components will use default values of `(0, 0, 1)`. The color type (`SFLOAT`, `UINT`, `SINT`) and bit width should also match the type of the shader input. See the following examples:

* `glm::IVec2` &ndash; `vk::Format::R32G32_SINT`, a 2-component vector of `i32`s
* `glm::UVec4` &ndash; `vk::Format::R32G32B32A32_UINT`, a 4-component vector of `u32`s
* `f64` &ndash; `vk::Format::R64_SFLOAT`, a double-precision (64-bit) float

The `format` parameter implicitly defines the byte size of attribute data and the `offset` parameter specifies the number of bytes since the start of the per-vertex data to read from. The binding is loading one `Vertex` at a time and the position attribute (`pos`) is at an offset of `0` bytes from the beginning of this struct.

```rust,noplaypen
let color = vk::VertexInputAttributeDescription::builder()
    .binding(0)
    .location(1)
    .format(vk::Format::R32G32B32_SFLOAT)
    .offset(size_of::<glm::Vec2>() as u32)
    .build();
```

The color attribute is described in much the same way.

Lastly, construct the array to return from the helper method:

```rust,noplaypen
[pos, color]
```

## Pipeline vertex input

We now need to set up the graphics pipeline to accept vertex data in this format by referencing the structures in `create_pipeline`. Find the `vertex_input_state` struct and modify it to reference the two descriptions:

```rust,noplaypen
let binding_descriptions = &[Vertex::binding_description()];
let attribute_descriptions = Vertex::attribute_descriptions();
let vertex_input_state = vk::PipelineVertexInputStateCreateInfo::builder()
    .vertex_binding_descriptions(binding_descriptions)
    .vertex_attribute_descriptions(&attribute_descriptions);
```

The pipeline is now ready to accept vertex data in the format of the `vertices` container and pass it on to our vertex shader. If you run the program now with validation layers enabled, you'll see that it complains that there is no vertex buffer bound to the binding. The next step is to create a vertex buffer and move the vertex data to it so the GPU is able to access it.
