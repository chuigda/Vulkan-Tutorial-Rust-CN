# 介绍

> 原文链接：<https://kylemayes.github.io/vulkanalia/introduction.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

本教程是用 Rust 对 <https://vulkan-tutorial.com> 的改写版本，应当归功于原教程的作者 ([Alexander Overvoorde](https://github.com/Overv)) 及[其他贡献者们](https://github.com/Overv/VulkanTutorial/graphs/contributors)。

同时，本教程也包含由笔者原创的章节（从 `Push Constants` 章节开始）。这些章节介绍了在几乎所有的 Vulkan 应用中都非常重要的 Vulkan 概念和特性。然而，正如这些章节中所说明的那样，这些特性是实验性的。

## 关于

本教程会教授一些 [Vulkan](https://khronos.org/vulkan) 图形与计算 API 的基础知识。Vulkan 是一个由 [Khronos 组织](https://www.khronos.org/) （因 OpenGL 而为人所知）提出的新 API，针对现代显卡的特性提供了更好的抽象。新的接口可以让你更好地描述你的应用程序要做什么，从而带来相比于诸如 [OpenGL](https://en.wikipedia.org/wiki/OpenGL) 和 [Direct3D](https://en.wikipedia.org/wiki/Direct3D) 之类的现有的图形 API 更好的性能和更少的意外驱动程序行为。Vulkan 的设计思想与 [Direct3D 12](https://en.wikipedia.org/wiki/Direct3D#Direct3D_12) 和 [Metal](https://en.wikipedia.org/wiki/Metal_(API)) 的思路相似，但 Vulkan 在跨平台方面具有优势，可以让你同时开发 Windows，Linux 和 Android 应用程序（并借由 [MoltenVK](https://github.com/KhronosGroup/MoltenVK) 开发 iOS 与 MacOS 应用程序）。

然而，为了这些增益，你所付出的代价便是你要使用一个更加冗长的 API。每个和图形 API 相关的细节都需要在你的应用程序中从头开始设置，包括最初的帧缓冲（framebuffer）创建以及缓冲区和纹理图像一类对象的内存管理。图形驱动程序会做更少手把手的指导，也就意味着你要在你的应用程序中做更多工作来确保正确的行为。

简而言之，Vulkan 并不是适合所有人使用的 API。它面向的是那些热衷于高性能计算机图形学，并且愿意为其投入精力的程序员们。如果你更感兴趣的是游戏开发而不是计算机图形学，那么你可能还是应该坚持使用 OpenGL 或者 Direct3D，因为它们不会那么快被 Vulkan 取代。另一个选择是使用像 [Unreal Engine](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_4) 或者 [Unity](https://en.wikipedia.org/wiki/Unity_(game_engine)) 这样的引擎，它们可以使用 Vulkan，但向你暴露一个更高层次的 API。

抛开上面的问题，让我们来看看跟着这个教程学习所需的一些东西：

* 一张支持 Vulkan 的显卡和驱动程序（[NVIDIA](https://developer.nvidia.com/vulkan-driver)，[AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan)，[Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540)）
* 使用 Rust 的经验
* Rust 1.51 或更高版本
* 一些的 3D 计算机图形学知识

本教程不会要求你有 OpenGL 或者 Direct3D 的知识储备，但要求你必须理解 3D 计算机图形学的基础。例如，教程中不会解释透视投影背后的数学原理。[这本在线书籍](https://paroj.github.io/gltut/)是一个很好的计算机图形学入门资源。其他一些很好的计算机图形学资源包括：

* [Ray tracing in one weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)
* [Physically Based Rendering book](http://www.pbr-book.org/)
* 在真正的开源游戏引擎中使用 Vulkan：[Quake](https://github.com/Novum/vkQuake) 和 [DOOM 3](https://github.com/DustinHLand/vkDOOM3)

如果你想要 C++ 教程，请查看原教程：<br/><https://vulkan-tutorial.com>

本教程由 [`vulkanalia`](https://github.com/KyleMayes/vulkanalia) crate 来提供 Rust 对 Vulkan API 的访问。`vulkanalia` 形成了对 Vulkan API 的原始绑定，同时也提供了一个轻量级的封装，使得这些 API 的使用更简单，也“更 Rust”（下一章里你会看到的）。也就是说，你不必费心考虑你的 Rust 程序如何与 Vulkan API 交互，同时你也能免受 Vulkan API 的危险性和冗长性的影响。

如果你想要一个使用更安全、细致封装的 [`vulkano`](https://vulkano.rs) crate 的 Rust Vulkan 教程，请查看这个教程：<https://github.com/bwasty/vulkan-tutorial-rs>。

## 教程结构

我们首先会速览一下 Vulkan 是如何工作的，以及要把第一个三角形画到屏幕上所需的工作。在你了解这些小的步骤在整个过程中的如何作用之后，其目的会更加清晰。接下来，我们会使用 [Vulkan SDK](https://lunarg.com/vulkan-sdk/) 来设置开发环境。

在这之后，我们会实现渲染你第一个三角形的 Vulkan 程序所需的所有基本组件。每一章的结构大致如下：

* 引入一个新的概念及其目的
* 使用所有相关的 API 调用，将其集成到你的程序中
* 将其抽象为辅助函数

尽管每一章都是作为前一章的后续章节编写的，但把每一章作为单独的介绍一个特定 Vulkan 特性的文章来阅读也是可以的。也就是说这个网站也可以作为一个 Vulkan 参考。所有 Vulkan 函数和类型都链接到了 Vulkan 规范或者 `vulkanalia` 的文档，你可以点击链接来了解更多。Vulkan 仍然是一个非常年轻的 API，所以 Vulkan 的规范本身可能有一些缺点。你可以提交反馈到 [这个 Khronos 仓库](https://github.com/KhronosGroup/Vulkan-Docs)。

如同前面所提到的，Vulkan API 是一个非常冗长的 API，提供了许多参数，能给你对图形硬件最大的控制。这就导致像创建纹理这种基本操作都要经过很多步骤，而且每次都要重复。因此，我们会在整个教程中会创建一系列自定义的辅助函数。

每一章也包含了一个链接，指向了该章节完成后的最终代码。如果你对代码的结构有任何疑问，或者你遇到了 bug，想要对比一下，你可以参考这些代码。

本教程旨在成为社区的共同努力。 Vulkan 仍然是一个相当新的 API，而最佳实践已经完全建立。 如果您对教程或者网站本身有任何类型的反馈，请随时向 [GitHub 存储库](https://github.com/KyleMayes/vulkanalia) 提交问题或拉取请求。

在你完成了第一个 Vulkan 三角形的绘制之后，我们会开始扩展程序，包括线性变换、纹理和 3D 模型。

如果你有使用其他图形 API 的经验，你会明白在第一个三角形在屏幕上显示出来之前，可能会有很多步骤。在 Vulkan 中也有很多这样的步骤，但是你会发现每个步骤都很容易理解，而且不会感觉繁琐。一旦你有了这个基础的三角形，绘制完全贴图的 3D 模型并不需要太多额外的工作，并且在这之后的每一步都会给你带来更多增益。

如果你在阅读本教程时遇到了任何问题，请查阅 FAQ，看看你的问题和解决方案是否已经在里面列出。接下来，你可以在[原教程](https://vulkan-tutorial.com/)的对应章节的评论区中找到是否有人遇到了相同的问题（如果不是 Rust 特有的问题）。
