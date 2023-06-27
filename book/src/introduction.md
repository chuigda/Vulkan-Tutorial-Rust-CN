# 介绍

本教程是 <https://vulkan-tutorial.com> 用 Rust 的改写。本教程主要应归功于原教程的作者 ([Alexander Overvoorde](https://github.com/Overv)) 及[其他贡献者们](https://github.com/Overv/VulkanTutorial/graphs/contributors)。

本教程也包含若干额外的，本教程作者原创的章节（从 `Push Constants` 章节开始）。这些章节介绍了在几乎所有 Vulkan 应用中都非常重要的 Vulkan 概念和特性。然而，正如这些章节中所声明的那样，这些特性是实验性的。

## 关于

本教程会教给你使用 [Vulkan](https://khronos.org/vulkan) 图形与计算 API 的基础知识。Vulkan 是一个由 [Knronos 组织](https://www.khronos.org/) （因 OpenGL 而为人所知）提出的新 API，针对现代显卡的特性提供了更好的抽象。新的接口可以让你更好地描述你的应用程序要做什么，从而带来相比于诸如 [OpenGL](https://en.wikipedia.org/wiki/OpenGL) 和 [Direct3D](https://en.wikipedia.org/wiki/Direct3D) 之类的现有的图形 API 更好的性能和更少的意外的驱动程序行为。这些 Vulkan 的设计思想与 [Direct3D 12](https://en.wikipedia.org/wiki/Direct3D#Direct3D_12) 和 [Metal](https://en.wikipedia.org/wiki/Metal_(API)) 的思路想死，但 Vulkan 在跨平台方面具有优势，可以让你同时开发 Windows，Linux 和 Android 应用程序（并借由 [MoltenVK](https://github.com/KhronosGroup/MoltenVK) 开发 iOS 与 MacOS 应用程序）。

然而，为了这些增益，你所付出的代价便是你要使用一个显著地更加冗长的 API。每个和图形 API 相关的细节都需要在你的应用程序中从头开始设置，包括最初的帧缓冲（framebuffer）创建以及缓冲区和纹理图像一类对象的内存管理。图形驱动程序会做更少手把手的指导，也就意味着你要在你的应用程序中做更多工作来确保正确的行为。

简而言之，Vulkan 并不是适合所有人使用的 API。它面向的是那些热衷于高性能计算机图形学，并且愿意投入精力的程序员们。如果你更感兴趣的是游戏开发而不是计算机图形学，那么你可能还是应该坚持使用 OpenGL 或者 Direct3D，因为它们不会很快被 Vulkan 取代。另一个选择是使用像 [Unreal Engine](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_4) 或者 [Unity](https://en.wikipedia.org/wiki/Unity_(game_engine)) 这样的引擎，它们可以使用 Vulkan，但向你暴露一个更高层次的 API。

抛开上面的问题，让我们来看看跟着这个教程学习所需的一些东西：

* 一张支持 Vulkan 的显卡和驱动程序（[NVIDIA](https://developer.nvidia.com/vulkan-driver)，[AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan)，[Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540)）
* 使用 Rust 的经验
* Rust 1.51 或更高版本
* 一些的 3D 计算机图形学知识

本教程不会假设你有 OpenGL 或者 Direct3D 的知识，但要求你必须理解 3D 计算机图形学的基础。例如，教程中不会解释透视投影背后的数学原理。[这本在线书籍](https://paroj.github.io/gltut/)是一个很好的计算机图形学入门资源。其他一些很好的计算机图形学资源包括：

* [Ray tracing in one weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)
* [Physically Based Rendering book](http://www.pbr-book.org/)
* 在真正的开源游戏引擎中使用 Vulkan：[Quake](https://github.com/Novum/vkQuake) 和 [DOOM 3](https://github.com/DustinHLand/vkDOOM3)

如果你想要 C++ 教程，请查看原教程：<br/><https://vulkan-tutorial.com>

本教程使用 [`vulkanalia`](https://github.com/KyleMayes/vulkanalia) crate 来提供 Rust 对 Vulkan API 的访问。`vulkanalia` 提供了对 Vulkan API 的原始绑定，同时也提供了一个轻量级的封装，使得这些 API 的使用更简单，也“更 Rust”（下一章里你会看到的）。也就是说，你不必费心考虑你的 Rust 程序如何与 Vulkan API 交互，同时你也能免受 Vulkan API 的危险性和冗长性的影响。

如果你想要一个使用更安全、细致封装的 [`vulkano`](https://vulkano.rs) crate 的 Rust Vulkan 教程，请查看这个教程：<https://github.com/bwasty/vulkan-tutorial-rs>。

## 教程结构

<!-- 下播曲ing 

天之涯，海之角，知交半零落
一壶浊酒尽余欢，今宵别梦寒

天之涯，海之角，知交半零落
一壶浊酒尽余欢，今宵别梦寒

-->

We'll start with an overview of how Vulkan works and the work we'll have to do to get the first triangle on the screen. The purpose of all the smaller steps will make more sense after you've understood their basic role in the whole picture. Next, we'll set up the development environment with the [Vulkan SDK](https://lunarg.com/vulkan-sdk/).

After that we'll implement all of the basic components of a Vulkan program that are necessary to render your first triangle. Each chapter will follow roughly the following structure:

* Introduce a new concept and its purpose
* Use all of the relevant API calls to integrate it into your program
* Abstract parts of it into helper functions

Although each chapter is written as a follow-up on the previous one, it is also possible to read the chapters as standalone articles introducing a certain Vulkan feature. That means that the site is also useful as a reference. All of the Vulkan functions and types are linked to the either the Vulkan specification or to the `vulkanalia` documentation, so you can click them to learn more. Vulkan is still a fairly young API, so there may be some shortcomings in the specification itself. You are encouraged to submit feedback to [this Khronos repository](https://github.com/KhronosGroup/Vulkan-Docs).

As mentioned before, the Vulkan API has a rather verbose API with many parameters to give you maximum control over the graphics hardware. This causes basic operations like creating a texture to take a lot of steps that have to be repeated every time. Therefore we'll be creating our own collection of helper functions throughout the tutorial.

Every chapter will also start with a link to the final code for that chapter. You can refer to it if you have any doubts about the structure of the code, or if you're dealing with a bug and want to compare.

This tutorial is intended to be a community effort. Vulkan is still a fairly new API and best practices have been fully established. If you have any type of feedback on the tutorial and site itself, then please don't hesitate to submit an issue or pull request to the [GitHub repository](https://github.com/KyleMayes/vulkanalia).

After you've gone through the ritual of drawing your very first Vulkan powered triangle onscreen, we'll start expanding the program to include linear transformations, textures and 3D models.

If you've played with graphics APIs before, then you'll know that there can be a lot of steps until the first geometry shows up on the screen. There are many of these initial steps in Vulkan, but you'll see that each of the individual steps is easy to understand and does not feel redundant. It's also important to keep in mind that once you have that boring looking triangle, drawing fully textured 3D models does not take that much extra work, and each step beyond that point is much more rewarding.

If you encounter any problems while following the tutorial, check the FAQ to see if your problem and its solution is already listed there. Next, you might find someone who had the same problem (if it is not Rust-specific) in the comment section for the corresponding chapter in the [original tutorial](https://vulkan-tutorial.com/).
