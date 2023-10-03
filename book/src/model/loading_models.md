# 加载模型

> 原文链接：<https://kylemayes.github.io/vulkanalia/model/loading_models.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/27_model_loading.rs)

Your program is now ready to render textured 3D meshes, but the current geometry in the `vertices` and `indices` arrays is not very interesting yet. In this chapter we're going to extend the program to load the vertices and indices from an actual model file to make the graphics card actually do some work.

现在你的程序已经可以渲染带纹理的 3D 网格了，但当前 `vertices` 和 `indices` 数组中的几何图形有些乏味。在本章中，我们将扩展程序，从一个实际的模型文件中加载顶点和索引，让显卡可以实际地做点事情。

Many graphics API tutorials have the reader write their own OBJ loader in a chapter like this. The problem with this is that any remotely interesting 3D application will soon require features that are not supported by this file format, like skeletal animation. We *will* load mesh data from an OBJ model in this chapter, but we'll focus more on integrating the mesh data with the program itself rather than the details of loading it from a file.

许多图形 API 教程都会让读者在这样的章节中编写自己的 OBJ 加载器。但这样做的问题是，更有趣的 3D 应用程序需要许多功能，例如骨骼动画，而 OBJ 格式不支持这些功能。我们*将会*在本章中从 OBJ 模型加载网格数据，但我们将更多地关注将网格数据与程序本身集成，而不是从文件加载它的细节。

## 库

We will use the [`tobj`](https://crates.io/crates/tobj) crate to load vertices and faces from an OBJ file. If you followed the instructions in the `Development environment` chapter you already have this dependency installed and ready to go.

我们将使用 [`tobj`](https://crates.io/crates/tobj) crate 从 OBJ 文件中加载顶点和面。如果你遵照了“开发环境”那一章中的说明，那么这个依赖应该已经安装好并且可以使用了。

## 示例网格

In this chapter we won't be enabling lighting yet, so it helps to use a sample model that has lighting baked into the texture. An easy way to find such models is to look for 3D scans on [Sketchfab](https://sketchfab.com/). Many of the models on that site are available in OBJ format with a permissive license.

在本章中我们不会启用光照，所以最好使用一个纹理上已经烘焙好光照的示例模型。查找这样的模型的一个简单方法是在 [Sketchfab](https://sketchfab.com/) 上寻找 3D 扫描模型。该网站上的许多模型都以 OBJ 格式提供，并且有宽松的许可证。

For this tutorial I've decided to go with the [Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38) model by [nigelgoh](https://sketchfab.com/nigelgoh) ([CC BY 4.0](https://web.archive.org/web/20200428202538/https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38)). I tweaked the size and orientation of the model to use it as a drop in replacement for the current geometry:

在本教程中，我决定使用 [nigeloh](https://sketchfab.com/nigelgoh) 做的的 [Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38) 模型（[CC BY 4.0](https://web.archive.org/web/20200428202538/https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38)）。我调整了模型的大小和方向，以便将其用作当前几何图形的替代品：

* [viking_room.obj](../images/viking_room.obj)
* [viking_room.png](../images/viking_room.png)

> **Note:** The `.obj` and `.png` files that are included in this tutorial may not be identical to the original files. Make sure you use the files from this tutorial (and beyond that, the `.obj` and `.png` files may also differ from the files used in the [original C++ tutorial](https://vulkan-tutorial.com)).

> **注意：**本教程中包含的 `.obj` 和 `.png` 文件可能与原始文件不同（并且可能也与[原始 C++ 教程](https://vulkan-tutorial.com)中使用的文件不同）。请确保使用本教程中的文件。

Feel free to use your own model, but make sure that it only consists of one material and that is has dimensions of about 1.5 x 1.5 x 1.5 units. If it is larger than that, then you'll have to change the view matrix. Put the model file and texture image in the `resources` directory.

你也可以随便使用你自己的模型，但确保它只使用了一个材质（material），并且其尺寸大约为 1.5 x 1.5 x 1.5 个单位。如果它比这个大，那么你将不得不改变视图矩阵。将模型文件和纹理图像放在 `resources` 目录中。

Update `create_texture_image` to use this path variable:

更新 `create_texture_image` 以从这个路径加载图像：

```rust,noplaypen
let image = File::open("resources/viking_room.png")?;
```

To double-check your image file is correct, you can also add this code in `create_texture_image` right after decoding the file as a PNG image:

如果要二次确认你的图像文件是正确的，你还可以在 `create_texture_image` 中在解码 PNG 图像后添加以下代码：

```rust,noplaypen
if width != 1024 || height != 1024 || reader.info().color_type != png::ColorType::Rgba {
    panic!("Invalid texture image.");
}
```

## 加载顶点和索引

We're going to load the vertices and indices from the model file now, so you should remove the global `VERTICES` and `INDICES` arrays now. Replace them with `AppData` fields:

现在我们将从模型文件中加载顶点和索引，所以你现在应该删除全局的 `VERTICES` 和 `INDICES` 数组。用 `AppData` 的字段替换它们：

```rust,noplaypen
struct AppData {
    // ...
    vertices: Vec<Vertex>,
    indices: Vec<u32>,
    vertex_buffer: vk::Buffer,
    vertex_buffer_memory: vk::DeviceMemory,
    // ...
}
```

You'll also need to replace all the references to the global arrays with the new `AppData` fields.

你还需要用新的 `AppData` 字段替换对全局数组的所有引用。

You should change the type of the indices from `u16` to `u32`, because there are going to be a lot more vertices than 65,536. Remember to also change the `cmd_bind_index_buffer` parameter:

你应该将索引的类型从 `u16` 改为 `u32`，因为顶点数量会远超过 65,536。记得也要改变 `cmd_bind_index_buffer` 的参数：

```rust,noplaypen
device.cmd_bind_index_buffer(
    *command_buffer,
    data.index_buffer,
    0,
    vk::IndexType::UINT32,
);
```

You'll also need to update the size of the index buffer in `create_index_buffer`:

你还需要更新 `create_index_buffer` 中索引缓冲区的大小：

```rust,noplaypen
let size = (size_of::<u32>() * data.indices.len()) as u64;
```

Next we are going to need some more imports:

接着我们需要再导入一些东西：

```rust,noplaypen
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::io::BufReader;
```

We're now going to write a `load_models` function that uses this library to populate the `vertices` and `indices` fields with the vertex data from the mesh. It should be called somewhere before the vertex and index buffers are created:

我们现在要编写一个 `load_models` 函数，它将使用 `tobj` 库来从网格中获取顶点数据并填充 `vertices` 和 `indices` 字段。它应该在创建顶点和索引缓冲区之前的某个地方被调用：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        load_model(&mut data)?;
        create_vertex_buffer(&instance, &device, &mut data)?;
        create_index_buffer(&instance, &device, &mut data)?;
        // ...
    }
}

fn load_model(data: &mut AppData) -> Result<()> {
    Ok(())
}
```

A model is loaded into the crate's data structures by calling the `tobj::load_obj_buf` function:

调用 `tobj::load_obj_buf` 函数来将模型加载到 `tobj` crate 的数据结构中：

```rust,noplaypen
let mut reader = BufReader::new(File::open("resources/viking_room.obj")?);

let (models, _) = tobj::load_obj_buf(
    &mut reader,
    &tobj::LoadOptions { triangulate: true, ..Default::default() },
    |_| Ok(Default::default()),
)?;
```

An OBJ file consists of positions, normals, texture coordinates and faces. Faces consist of an arbitrary amount of vertices, where each vertex refers to a position, normal and/or texture coordinate by index. This makes it possible to not just reuse entire vertices, but also individual attributes.

OBJ 文件由位置、法线、纹理坐标和面组成。面由任意数量的顶点组成，其中每个顶点通过索引引用位置、法线和/或纹理坐标。这使得 OBJ 文件中的面不仅可以重用整个顶点，还可以重用顶点的单个属性。

`tobj::load_obj_buf` returns a `Vec` of models and a `Vec` of materials. We are not interested in the materials, only the models, so the returned materials are ignored with `_`.

`tobj::load_obj_buf` 返回一个模型的 `Vec` 和一个材质的 `Vec`。我们对材质不感兴趣，只对模型感兴趣，所以返回的材质被忽略了。

The second `tobj::load_obj_buf` parameter specifies options for processing the loaded models. The `triangulate` field we are setting to `true` ensures that the components of the loaded models are converted to triangles. This is important because our rendering code can only handle triangles. We don't need this for our Viking room model since its faces are already triangles but it might be necessary if you try using a different OBJ file.

`tobj::load_obj_buf` 的第二个参数制定了处理加载的模型的选项。我们将 `triangulate` 字段设置为 `true`，以确保加载的模型的组件被转换为三角形。这很重要，因为我们的渲染代码只能处理三角形。我们的 Viking room 模型不需要这个，因为它的面已经是三角形了，但如果你尝试使用不同的 OBJ 文件，这可能是必要的。

The third `tobj::load_obj_buf` parameter is a callback used to load any materials referenced in the OBJ file. As we are not interested in the materials, we just return an empty material.

`tobj::load_obj_buf` 的第三个参数是一个回调，用于加载 OBJ 文件中引用的材质。我们对材质不感兴趣，所以我们只返回一个空材质。

We're going to combine all of the faces in the file into a single model, so just iterate over all of the shapes:

我们将把文件中的所有面组合成一个模型，所以我们遍历所有模型：

```rust,noplaypen
for model in &models {
}
```

The triangulation feature has already made sure that there are three vertices per face, so we can now directly iterate over the vertices and dump them straight into our `vertices` vector:

三角化功能已经确保每个面有三个顶点，所以我们现在可以直接遍历顶点并将它们直接转储到我们的 `vertices` 向量中：

```rust,noplaypen
for model in &models {
    for index in &model.mesh.indices {
        let vertex = Vertex {
            pos: vec3(0.0, 0.0, 0.0),
            color: vec3(1.0, 1.0, 1.0),
            tex_coord: vec2(0.0, 0.0),
        };

        data.vertices.push(vertex);
        data.indices.push(data.indices.len() as u32);
    }
}
```

For simplicity, we will assume that every vertex is unique for now, hence the simple auto-increment indices. The `index` variable is used to look up the actual vertex attributes in the `positions` and `texcoords` arrays:

简单起见，我们现在假设每个顶点都是唯一的，因此简单地自增索引就行。`index` 变量用于在 `positions` 和 `texcoords` 数组中查找实际的顶点属性：

```rust,noplaypen
let pos_offset = (3 * index) as usize;
let tex_coord_offset = (2 * index) as usize;

let vertex = Vertex {
    pos: vec3(
        model.mesh.positions[pos_offset],
        model.mesh.positions[pos_offset + 1],
        model.mesh.positions[pos_offset + 2],
    ),
    color: vec3(1.0, 1.0, 1.0),
    tex_coord: vec2(
        model.mesh.texcoords[tex_coord_offset],
        model.mesh.texcoords[tex_coord_offset + 1],
    ),
};
```

Unfortunately the `attrib.vertices` array is an array of `float` values instead of something like `cgmath::Vector3<f32>`, so you need to multiply the index by `3`. Similarly, there are two texture coordinate components per entry. The offsets of `0`, `1` and `2` are used to access the X, Y and Z components, or the U and V components in the case of texture coordinates.

不幸的是，`attrib.vertices` 数组是一个 `float` 类型而不是 `cgmath::Vector3<f32>` 类型的数组，所以你需要将索引乘以 `3`。类似地，每个条目有两个纹理坐标分量。`0`、`1` 和 `2` 的偏移量用于访问 X、Y 和 Z 分量，或者在纹理坐标的情况下访问 U 和 V 分量。

You may want to start compiling your program in release mode from now on because loading the texture and model can be quite slow without optimizations. If you run your program now you should see something like the following:

你可能想从现在开始在 release 模式下编译你的程序，因为没有优化的情况下加载纹理和模型可能会非常慢。如果你现在运行你的程序，你应该会看到如下所示的东西：

![](../images/inverted_texture_coordinates.png)

Great, the geometry looks correct, but what's going on with the texture? The OBJ format assumes a coordinate system where a vertical coordinate of `0` means the bottom of the image, however we've uploaded our image into Vulkan in a top to bottom orientation where `0` means the top of the image. Solve this by flipping the vertical component of the texture coordinates:

太棒了，几何图形看起来是正确的，但纹理怎么了？OBJ 格式使用这样一个坐标系：垂直坐标 `0` 表示图像的底部。但是我们已经将图像以自上而下的方式上传到 Vulkan 中，其中 `0` 表示图像的顶部。我们通过翻转纹理坐标的垂直分量来解决这个问题：

```rust,noplaypen
tex_coord: vec2(
    model.mesh.texcoords[tex_coord_offset],
    1.0 - model.mesh.texcoords[tex_coord_offset + 1],
),
```

When you run your program again, you should now see the correct result:

当你再次运行你的程序时，你应该会看到正确的结果：

![](../images/drawing_model.png)

All that hard work is finally beginning to pay off with a demo like this!

所有这些辛苦的工作终于开始得到回报了！

## 定点去重

Unfortunately we're not really taking advantage of the index buffer yet. The `vertices` vector contains a lot of duplicated vertex data, because many vertices are included in multiple triangles. We should keep only the unique vertices and use the index buffer to reuse them whenever they come up. A straightforward way to implement this is to use a `HashMap` to keep track of the unique vertices and respective indices:

不幸的是，我们还没有真正地从索引缓冲中获益。`vertices` 现在包含了大量重复的顶点数据，因为许多顶点都被包含在多个三角形中。我们应该只保留唯一一个顶点，并使用索引缓冲区在它们出现时重用它们。要实现这一点，一种直接的方法是使用 `HashMap` 来跟踪唯一的顶点和相应的索引：

```rust,noplaypen
let mut unique_vertices = HashMap::new();

for model in &models {
    for index in &model.mesh.indices {
        // ...

        if let Some(index) = unique_vertices.get(&vertex) {
            data.indices.push(*index as u32);
        } else {
            let index = data.vertices.len();
            unique_vertices.insert(vertex, index);
            data.vertices.push(vertex);
            data.indices.push(index as u32);
        }
    }
```

Every time we read a vertex from the OBJ file, we check if we've already seen a vertex with the exact same position and texture coordinates before. If not, we add it to `vertices` and store its index in the `unique_vertices` container. After that we add the index of the new vertex to `indices`. If we've seen the exact same vertex before, then we look up its index in `unique_vertices` and store that index in `indices`.

我们从 OBJ 文件中读取一个索引，并检查我们之前是否已经看到过一个具有完全相同的位置和纹理坐标的顶点。如果没有，我们将它添加到 `vertices` 中，并将其索引存储在 `unique_vertices` 容器中。之后，我们将新顶点的索引添加到 `indices` 中。如果我们之前看到过完全相同的顶点，那么我们将在 `unique_vertices` 中查找它的索引，并将该索引存储在 `indices` 中。

The program will fail to compile right now because we need to implement the `Hash` trait for our `Vertex` struct to be able to use it as the key of a `HashMap`. Unfortunately since `Vertex` contains `f32`s we will need to implement `Hash` and the required traits (`PartialEq` and `Eq`) by hand (note that our `Eq` implementation is valid only as long as there aren't any `NaN`s in our vertex data which for now is a safe assumption).

程序现在将无法编译，因为我们需要为我们的 `Vertex` 结构实现 `Hash` trait，以便将其用作 `HashMap` 的键。不幸的是，由于 `Vertex` 包含 `f32`，我们需要手动实现 `Hash` 和所需的 trait（`PartialEq` 和 `Eq`）（注意，我们的 `Eq` 实现只在顶点数据中没有 `NaN` 的情况下才有效，这是一个安全的假设）。

```rust,noplaypen
impl PartialEq for Vertex {
    fn eq(&self, other: &Self) -> bool {
        self.pos == other.pos
            && self.color == other.color
            && self.tex_coord == other.tex_coord
    }
}

impl Eq for Vertex {}

impl Hash for Vertex {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.pos[0].to_bits().hash(state);
        self.pos[1].to_bits().hash(state);
        self.pos[2].to_bits().hash(state);
        self.color[0].to_bits().hash(state);
        self.color[1].to_bits().hash(state);
        self.color[2].to_bits().hash(state);
        self.tex_coord[0].to_bits().hash(state);
        self.tex_coord[1].to_bits().hash(state);
    }
}
```

You should now be able to successfully compile and run your program. If you check the size of `vertices`, then you'll see that it has shrunk down from 1,500,000 to 265,645! That means that each vertex is reused in an average number of ~6 triangles. This definitely saves us a lot of GPU memory.

现在你应该能够成功编译和运行你的程序。如果你检查 `vertices` 的大小，你会发现它已经从 1,500,000 减少到 265,645！这意味着每个顶点平均被大约 6 个三角形重用。这绝对节省了大量的 GPU 内存。
