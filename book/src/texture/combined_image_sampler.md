# 组合图像采样器

> 原文链接：<https://kylemayes.github.io/vulkanalia/texture/combined_immage_sampler.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/25_texture_mapping.rs) | [shader.vert](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/25/shader.vert) | [shader.frag](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/shaders/25/shader.frag)

我们在 uniform 缓冲部分第一次认识了描述符。在本章中我们将看到一种新的描述符：*组合图像采样器*。这种描述符使得着色器可以通过像我们在上一章中创建的采样器对象来访问图像资源。

首先我们修改描述符集合布局、描述符池以及描述符集合，使其包含一个组合图像采样器描述符。然后我们将在 `Vertex` 中添加纹理坐标，并修改片元着色器，使其从纹理中读取颜色而不是仅仅插值顶点颜色。

## 更新描述符

在 `create_descriptor_set_layout` 函数中为组合图像采样器描述符添加一个 `vk::DescriptorSetLayoutBinding`。我们将其放在 uniform 缓冲之后的绑定中：

```rust,noplaypen
let sampler_binding = vk::DescriptorSetLayoutBinding::builder()
    .binding(1)
    .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
    .descriptor_count(1)
    .stage_flags(vk::ShaderStageFlags::FRAGMENT);

let bindings = &[ubo_binding, sampler_binding];
let info = vk::DescriptorSetLayoutCreateInfo::builder()
    .bindings(bindings);
```

记得将 `stage_flags` 设为 `vk::ShaderStageFlags::FRAGMENT`，以指示我们将会在片元着色器中使用组合图像采样器描述符 —— 这是片元的颜色将会被确定的地方。在顶点着色器中使用纹理采样是可能的，例如通过[高度图](https://en.wikipedia.org/wiki/Heightmap)动态地改变顶点网格的形状。

我们必须创建一个更大的描述符池，在 `vk::DescriptorPoolCreateInfo` 中添加另一个 `vk::DescriptorType::COMBINED_IMAGE_SAMPLER` 类型的 `vk::DescriptorPoolSize`，从而为组合图像采样器的分配腾出空间。转到 `create_descriptor_pool` 函数，为组合图像采样器描述符添加一个 `vk::DescriptorPoolSize`：

```rust,noplaypen
let sampler_size = vk::DescriptorPoolSize::builder()
    .type_(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
    .descriptor_count(data.swapchain_images.len() as u32);

let pool_sizes = &[ubo_size, sampler_size];
let info = vk::DescriptorPoolCreateInfo::builder()
    .pool_sizes(pool_sizes)
    .max_sets(data.swapchain_images.len() as u32);
```

有一些问题是校验层无法捕获的，描述符池不够大就是一个很好的例子：从 Vulkan 1.1 开始，如果描述符池不够大，`allocate_descriptor_sets` 可能会失败并返回错误码 `vk::ErrorCode::OUT_OF_POOL_MEMORY`，但驱动程序也可能会尝试在内部解决这个问题。这意味着有时（取决于硬件、池大小和分配大小）驱动程序会允许我们超出描述符池的限制。其他时候，`allocate_descriptor_sets` 将会失败并返回 `vk::ErrorCode::OUT_OF_POOL_MEMORY`。如果分配在某些机器上成功，但在其他机器上失败，这可能会让人特别沮丧。

因为 Vulkan 把分配的责任转移给了驱动程序，所以只分配与描述符池创建时相应的 `descriptor_count` 成员指定的某种类型的描述符（`vk::DescriptorType::COMBINED_IMAGE_SAMPLER` 等）不再是一个严格的要求。然而，最好还是这样做。并且在将来，如果你启用了[最佳实践校验](https://vulkan.lunarg.com/doc/view/1.1.126.0/windows/best_practices.html)，`VK_LAYER_KHRONOS_validation` 将会对这种类型的问题发出警告。

最后一步就是将实际的图像和采样器绑定到描述符集合中的描述符上了。转到 `create_descriptor_sets` 函数。组合图像采样器结构的资源必须在 `vk::DescriptorImageInfo` 结构中指定，就像 uniform 缓冲描述符的缓冲资源在 `vk::DescriptorBufferInfo` 结构中指定一样。这就是前一章中的对象结合在一起的地方。

```rust,noplaypen
let info = vk::DescriptorImageInfo::builder()
    .image_layout(vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL)
    .image_view(data.texture_image_view)
    .sampler(data.texture_sampler);

let image_info = &[info];
let sampler_write = vk::WriteDescriptorSet::builder()
    .dst_set(data.descriptor_sets[i])
    .dst_binding(1)
    .dst_array_element(0)
    .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
    .image_info(image_info);

device.update_descriptor_sets(
    &[ubo_write, sampler_write],
    &[] as &[vk::CopyDescriptorSet],
);
```

描述符需要使用图像信息更新。这一次我们使用 `image_info` 数组，而不是 `buffer_info`。描述符现在已经可以被着色器使用了！

## 纹理坐标

纹理映射还缺少一个重要的组成部分，那就是每个顶点的实际坐标。这些坐标决定了图像如何映射到几何体上。

```rust,noplaypen
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Vertex {
    pos: Vec2,
    color: Vec3,
    tex_coord: Vec2,
}

impl Vertex {
    const fn new(pos: Vec2, color: Vec3, tex_coord: Vec2) -> Self {
        Self { pos, color, tex_coord }
    }

    fn binding_description() -> vk::VertexInputBindingDescription {
        vk::VertexInputBindingDescription::builder()
            .binding(0)
            .stride(size_of::<Vertex>() as u32)
            .input_rate(vk::VertexInputRate::VERTEX)
            .build()
    }

    fn attribute_descriptions() -> [vk::VertexInputAttributeDescription; 3] {
        let pos = vk::VertexInputAttributeDescription::builder()
            .binding(0)
            .location(0)
            .format(vk::Format::R32G32_SFLOAT)
            .offset(0)
            .build();
        let color = vk::VertexInputAttributeDescription::builder()
            .binding(0)
            .location(1)
            .format(vk::Format::R32G32B32_SFLOAT)
            .offset(size_of::<Vec2>() as u32)
            .build();
        let tex_coord = vk::VertexInputAttributeDescription::builder()
            .binding(0)
            .location(2)
            .format(vk::Format::R32G32_SFLOAT)
            .offset((size_of::<Vec2>() + size_of::<Vec3>()) as u32)
            .build();
        [pos, color, tex_coord]
    }
}
```

修改 `Vertex` 结构体并为纹理坐标添加一个 `Vec2` 类型的字段。记得也添加一条 `vk::VertexInputAttributeDescription`，这样我们就能在顶点着色器中访问纹理坐标了。这样我们才能将它们从顶点着色器传递给片元着色器，以便在正方形表面上进行插值。

```rust,noplaypen
static VERTICES: [Vertex; 4] = [
    Vertex::new(vec2(-0.5, -0.5), vec3(1.0, 0.0, 0.0), vec2(1.0, 0.0)),
    Vertex::new(vec2(0.5, -0.5), vec3(0.0, 1.0, 0.0), vec2(0.0, 0.0)),
    Vertex::new(vec2(0.5, 0.5), vec3(0.0, 0.0, 1.0), vec2(0.0, 1.0)),
    Vertex::new(vec2(-0.5, 0.5), vec3(1.0, 1.0, 1.0), vec2(1.0, 1.0)),
];
```

在本教程中，我会简单地使用左上角为 `0, 0`，右下角为 `1, 1` 的坐标填充正方形。你可以随意尝试不同的坐标，并试着使用小于 `0` 或大于 `1` 的坐标来看看寻址模式的效果！

## 着色器

最后一步就是修改着色器，使其从纹理中采样出颜色。 我们首先需要修改顶点着色器，将纹理坐标传递给片元着色器：

```glsl
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

和逐顶点颜色一样，`fragTexCoord` 值也会被光栅化器在正方形区域内平滑插值。我们可以通过让片元着色器将纹理坐标作为颜色输出来可视化这一点：

```glsl
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

别忘记重新编译着色器！之后，你应该会看到下面的图像。

![](../images/texcoord_visualization.png)

绿色通道代表了水平坐标，红色通道代表了垂直坐标。黑色和黄色的角落证实了纹理坐标在正方形上从 `0, 0` 插值到 `1, 1` 的正确性。使用颜色可视化数据是着色器编程中的 `printf` 调试等效方法，因为没有更好的选择！

组合图像采样器描述符在 GLSL 中是使用采样器 uniform 表示的。在片元着色器中添加一个引用：

```glsl
layout(binding = 1) uniform sampler2D texSampler;
```

对于其他类型的图像，有等效的 `sampler1D` 和 `sampler3D` 类型。记得在这里使用正确的绑定。

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

使用内建的 `texture` 函数采样纹理。`texture` 函数接受一个 `sampler` 和一个坐标作为参数。采样器会自动处理过滤和变换。当你运行应用程序时，你应该会在正方形上看到纹理：

![](../images/texture_on_square.png)

试着缩放纹理坐标，得到大于 `1` 的值，来试验寻址模式的功能。例如，当使用 `vk::SamplerAddressMode::REPEAT` 时，下面的片元着色器会产生下面的图像：

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![](../images/texture_on_square_repeated.png)

你也可以用顶点颜色来改变纹理颜色：

```glsl
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

我在这里分离了 RGB 和 alpha 通道，以免缩放 alpha 通道。

![](../images/texture_on_square_colorized.png)

现在你知道如何在着色器中访问图像了！当与帧缓冲中也写入的图像结合使用时，这是一种非常强大的技术。你可以使用这些图像作为输入来实现很酷的效果，比如后处理和 3D 世界中的相机显示。
