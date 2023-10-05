# 描述顶点输入

> 原文链接：<https://kylemayes.github.io/vulkanalia/vertex/vertex_input_description.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/17_vertex_input.rs) | [shader.vert](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/17/shader.vert) | [shader.frag](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/17/shader.frag)

在接下来的几章中，我们将用内存中的顶点缓冲替换顶点着色器中硬编码的顶点数据。我们将从最简单的方法开始，即创建一个对 CPU 可见的缓冲，并直接将顶点数据复制到其中。之后，我们将学习如何使用暂存缓冲将顶点数据复制到高性能内存中。

## 顶点着色器

首先修改顶点着色器，不再在着色器代码本身中包含顶点数据。顶点着色器将使用 `in` 关键字从顶点缓冲中获取输入。

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

`inPosition` 和 `inColor` 变量是*顶点属性*。它们是在顶点缓冲中为每个顶点指定的属性，就像我们之前手动使用两个数组为每个顶点指定了位置和颜色一样。记得重新编译顶点着色器！

与 `fragColor` 类似，`layout(location = x)` 注解为输入变量分配了索引，我们稍后可以用索引来引用这些变量。重要的是要知道，某些类型（例如 64 位的 `dvec3` 向量）使用多个*槽位*。这意味着在它之后的索引必须至少 +2：

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

你可以在 [OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)) 中找到关于布局限定符的更多信息。

## 顶点数据

我们将从着色器代码中将顶点数据移到程序代码的数组中。首先，向我们的程序添加几个导入项和一些类型别名。

```rust,noplaypen
use std::mem::size_of;

use cgmath::{vec2, vec3};

type Vec2 = cgmath::Vector2<f32>;
type Vec3 = cgmath::Vector3<f32>;
```

`size_of` 函数将用于计算我们将要定义的顶点数据的大小，而 `cgmath` 则定义了我们需要的向量类型。

接下来，创建一个名为 `Vertex` 的 `#[repr(C)]` 结构体，其中包含我们将在顶点着色器中使用的两个属性，并添加一个简单的构造函数：

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Vertex {
    pos: Vec2,
    color: Vec3,
}

impl Vertex {
    const fn new(pos: Vec2, color: Vec3) -> Self {
        Self { pos, color }
    }
}
```

`cgmath` 提供了与着色器语言中使用的向量类型完全匹配的 Rust 类型。

```rust,noplaypen
static VERTICES: [Vertex; 3] = [
    Vertex::new(vec2(0.0, -0.5), vec3(1.0, 0.0, 0.0)),
    Vertex::new(vec2(0.5, 0.5), vec3(0.0, 1.0, 0.0)),
    Vertex::new(vec2(-0.5, 0.5), vec3(0.0, 0.0, 1.0)),
];
```

现在我们可以使用 `Vertex` 结构体来定义顶点数据了。我们使用了与之前完全相同的位置和颜色值，但现在顶点位置和颜色数据被合并进一个顶点数组中。这种定义顶点数据的方式也被称为*交错顶点属性*（interleaving vertex attributes）。

## 绑定描述

接下来的步骤是告诉 Vulkan 在顶点数据上传到 GPU 内存后如何将这些数据传递给顶点着色器。我们需要两种结构体来传递这些信息。

第一种结构体是顶点绑定 `vk::VertexInputBindingDescription`，我们为 `Vertex` 结构体添加一个方法来填充它：

```rust,noplaypen
impl Vertex {
    fn binding_description() -> vk::VertexInputBindingDescription {
    }
}
```

<!-- 我真不知道这里这个 rate 是啥，但至少跟速率毫无关系 -->
顶点绑定描述了从内存中加载数据的方式：它指定了数据条目之间的字节数，以及是在每个顶点后移动到下一个数据条目，还是在每个实例后移动到下一个数据条目。

```rust,noplaypen
vk::VertexInputBindingDescription::builder()
    .binding(0)
    .stride(size_of::<Vertex>() as u32)
    .input_rate(vk::VertexInputRate::VERTEX)
    .build()
```

我们的每个顶点的数据都被紧密地打包在一个数组中，因此我们只会有一个绑定。`binding` 参数指定绑定在绑定数组中的索引。`stride` 参数指定从一个条目到下一个条目的字节数，而 `input_rate` 参数可以有以下值之一：

* `vk::VertexInputRate::VERTEX` &ndash; 在每个顶点后移动到下一个数据条目
* `vk::VertexInputRate::INSTANCE` &ndash; 在每个实例后移动到下一个数据条目

由于我们不会使用实例化渲染（instanced rendering），因此我们将使用每个顶点的数据。

## 属性描述

第二种结构体是 `vk::VertexInputAttributeDescription`，它用于描述顶点输入。我们将为 `Vertex` 结构体添加另一个辅助方法来填充这些结构。

```rust,noplaypen
impl Vertex {
    fn attribute_descriptions() -> [vk::VertexInputAttributeDescription; 2] {
    }
}
```

如函数原型所示，我们将会创建两个这样的结构体。这个结构体描述了如何从顶点绑定产生的顶点数据块中提取顶点属性。我们有两个属性：位置和颜色，因此我们需要两个结构体。

```rust,noplaypen
let pos = vk::VertexInputAttributeDescription::builder()
    .binding(0)
    .location(0)
    .format(vk::Format::R32G32_SFLOAT)
    .offset(0)
    .build();
```

`binding` 参数告诉 Vulkan 顶点数据来自哪个绑定。`location` 参数引用了顶点着色器中输入的 `location` 指令。顶点着色器中顶点位置的 `location` 是 `0`，它有两个 32 位浮点分量。

`format` 参数描述属性的数据类型。有点混淆的是，`format` 字段使用与颜色格式相同的枚举类型。以下是常见的着色器类型和对应的颜色格式枚举：

* `f32` &ndash; `vk::Format::R32_SFLOAT`&nbsp;
* `cgmath::Vector2<f32>` (我们的 `Vec2`) &ndash; `vk::Format::R32G32_SFLOAT`&nbsp;
* `cgmath::Vector3<f32>` (我们的 `Vec3`) &ndash; `vk::Format::R32G32B32_SFLOAT`&nbsp;
* `cgmath::Vector4<f32>` &ndash; `vk::Format::R32G32B32A32_SFLOAT`&nbsp;

如你所见，颜色格式的颜色通道数量应与着色器数据类型的分量数量相匹配。颜色格式的通道数量比着色器数据类型的分量数量多也是可以的，但多余的通道将被静默丢弃。如果通道数量少于分量数量，则 BGA 分量将使用默认值 `(0, 0, 1)` 。颜色类型（ `SFLOAT` ，`UINT` ，`SINT` ）和位宽度也应与着色器数据类型匹配。请参阅以下示例：

* `cgmath::Vector2<i32>` &ndash; `vk::Format::R32G32_SINT`，包含 `i32` 的 2 分量向量
* `cgmath::Vector4<u32>` &ndash; `vk::Format::R32G32B32A32_UINT`，包含 `u32` 的 4 分量向量
* `f64` &ndash; `vk::Format::R64_SFLOAT`，双精度（64位）浮点数

`format` 参数隐式地定义了属性数据的字节大小，而 `offset` 参数指定从顶点数据开始的字节数：绑定每次加载一个 `Vertex`，位置属性（`pos`）从该结构体的开始处偏移 `0` 字节。

```rust,noplaypen
let color = vk::VertexInputAttributeDescription::builder()
    .binding(0)
    .location(1)
    .format(vk::Format::R32G32B32_SFLOAT)
    .offset(size_of::<Vec2>() as u32)
    .build();
```

颜色属性的描述方式基本相同。

最后，构造要从辅助方法返回的数组：

```rust,noplaypen
[pos, color]
```

## 管线顶点输入

现在我们需要在 `create_pipeline` 中引用这两个描述结构体，以让图形管线接受这种格式的顶点数据。找到 `vertex_input_state` 结构并修改它，让它引用这两个描述结构体：

```rust,noplaypen
let binding_descriptions = &[Vertex::binding_description()];
let attribute_descriptions = Vertex::attribute_descriptions();
let vertex_input_state = vk::PipelineVertexInputStateCreateInfo::builder()
    .vertex_binding_descriptions(binding_descriptions)
    .vertex_attribute_descriptions(&attribute_descriptions);
```

现在，管线已准备好接受 `vertices` 容器格式的顶点数据，并将其传递给我们的顶点着色器。如果你现在启用了校验层并运行程序，你会看到它抱怨没有顶点缓冲被绑定到绑定点。接下来的步骤是创建一个顶点缓冲，并将顶点数据移到其中，以便 GPU 能够访问它。
