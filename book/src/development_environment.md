# 开发环境

> 原文链接：<https://kylemayes.github.io/vulkanalia/development_environment.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

在这个章节中，我们将会安装 Vulkan SDK 并搭建开发 Vulkan 应用所需的环境。此教程假设你已经有一个搭建好的 Rust（1.70+）开发环境。

## Cargo 项目

首先，我们创建一个 Cargo 项目：

`cargo new vulkan-tutorial`

在执行这个命令后，你会看到一个叫做 `vulkan-tutorial` 的文件夹，里面有一个简单的生成 Rust 可执行文件的 Cargo 项目。

打开这个文件夹里的 `Cargo.toml` 文件，并且将下列依赖加入其中的 `[dependencies]` 部分：

```toml
anyhow = "1"
log = "0.4"
cgmath = "0.18"
nalgebra-glm = "0.18"
png = "0.17"
pretty_env_logger = "0.5"
thiserror = "1"
tobj = { version = "3", features = ["log"] }
vulkanalia = { version = "=0.25.0", features = ["libloading", "provisional", "window"] }
winit = "0.29"
```

* `anyhow` &ndash; 用于简单的错误处理
* `log` &ndash; 日志库
* `cgmath` &ndash; 一个 Rust 语言的 [GLM](https://glm.g-truc.net/0.9.9/index.html)（graphics math library，图形数学库）替代
* `png` &ndash; 用于将 PNG 图片文件加载到纹理
* `pretty_env_logger` &ndash; 用于打印日志到控制台
* `thiserror` &ndash; 用于在自定义错误类型时减少样板代码
* `tobj` &ndash; 用于加载 [Wavefront .obj 格式](https://en.wikipedia.org/wiki/Wavefront_.obj_file) 的 3D 模型
* `vulkanalia` &ndash; 用于调用 Vulkan API
* `winit` &ndash; 用于创建将进行渲染的窗口

## Vulkan SDK

在开发 Vulkan 应用时需要用到的最关键的组件就是 Vulkan SDK。它包含了头文件、标准校验层、调试工具，以及一个 Vulkan 函数加载器。加载器将会在运行时从驱动中寻找 Vulkan 函数，如果你熟悉 OpenGL 的话，它的功能与 GLEW 类似。

### Windows

SDK 能在 [LunarG 网站](https://vulkan.lunarg.com/)下载。创建账户不是必须的，但它会给你阅读一些额外文档的权限，这些文档或许对你有用。

![](./images/vulkan_sdk_download_buttons.png)

继续完成安装，并且注意 SDK 的安装路径。我们需要做的第一件事就是验证你的显卡与驱动支持 Vulkan。进入 SDK 的安装路径，打开 `Bin` 文件夹并且运行 `vkcube.exe` 示例应用。你应该会看到这个画面：

![](./images/cube_demo.png)

如果你收到了一条错误信息，那你需要确保你的显卡驱动是最新的，包含 Vulkan 运行时，并且你的显卡支持 Vulkan。主流品牌的驱动下载链接详见[介绍章节](introduction.html)。

这个文件夹里有另外两个对开发很有用的程序。`glslangValidator.exe` 和 `glslc.exe` 将会把人类可阅读的 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language) （OpenGL Shading Language，OpenGL 着色器语言）代码编译为字节码。我们将会在[着色器模块章节](pipeline/shader_modules.html)深入讨论这部分内容。`Bin` 文件夹也包含了 Vulkan 加载器与校验层的二进制文件；`Lib` 文件夹则包含了库。

你可以自由地探索其它文件，但本教程并不会用到它们。

### Linux

以下操作说明面向 Ubuntu 用户，非 Ubuntu 用户也可以将 `apt` 命令换成合适的你使用的包管理器的命令。

在 Linux 上开发 Vulkan 应用时需要用到的最关键的组件是 Vulkan 加载器，校验层，以及一些用来测试你的机器是否支持 Vulkan 的命令行实用工具：

* `sudo apt install vulkan-tools` &ndash; 命令行实用工具，最关键的两个是 `vulkaninfo` 和 `vkcube`。运行这两个命令来测试你的机器是否支持 Vulkan。
* `sudo apt install libvulkan-dev` &ndash; 安装 Vulkan 加载器。加载器将会在运行时从驱动中寻找这些函数，如果你熟悉 OpenGL 的话，它的功能与 GLEW 类似。
* `sudo apt install vulkan-validationlayers-dev` &ndash; 安装标准校验层。这在调试 Vulkan 应用程序时非常关键，我们会在之后的章节中讨论这部分内容。

如果你安装成功了，你在 Vulkan 部分没有别的需要做的了。记得运行 `vkcube` 并确保你可以在一个窗口中看见这个画面：

![](./images/cube_demo_nowindow.png)

如果你收到了一条错误信息，那你需要确保你的显卡驱动是最新的，包含 Vulkan 运行时，并且你的显卡支持 Vulkan。主流品牌的驱动下载链接详见[介绍章节](introduction.html)。

### MacOS

SDK 可在 [LunarG 网站](https://vulkan.lunarg.com/) 下载。创建账户不是必须的，但它会给你阅读一些或许对你有用的额外文档的权限。

![](./images/vulkan_sdk_download_buttons.png)

MacOS 版本的 SDK 在内部使用了 [MoltenVK](https://moltengl.com/)。Vulkan 在 MacOS 上没有原生支持，所以 MoltenVK 会作为中间层把 Vulkan API 的调用翻译至苹果的 Metal 图形框架。这样你就可以享受到苹果的 Metal 框架在调试与性能上的优点。

下载完成之后，将其解压到你自己选择的文件夹。在解压后的文件夹内，你可以在 `Applications` 文件夹中找到一些使用 SDK 运行的示例应用的可执行文件。运行 `vkcube` 示例应用，你会看到这个画面：

![](./images/cube_demo_mac.png)

#### 环境配置

当在 Vulkan SDK 目录以外的地方运行 Vulkan 应用程序时，你可能会需要运行由 Vulkan SDK 提供的 `setup-env.sh` 脚本，以避免找不到 Vulkan 库（例如 `libvulkan.dylib`）导致的问题。如果你把 Vulkan SDK 安装到了默认位置，脚本应该可以在如下目录中找到：`~/VulkanSDK/1.3.280.1/setup-env.sh`（用你实际安装的版本号替换 `1.3.280.1` 以匹配你的 Vulkan 安装）。

你也可以把这个脚本添加到你 Shell 的启动脚本中，这样它就能自动运行了。例如你可以在 `~/.zshrc` 文件中添加这样一行：

```
source ~/VulkanSDK/1.3.280.1/setup-env.sh
```
