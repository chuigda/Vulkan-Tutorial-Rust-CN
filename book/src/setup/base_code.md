# 基础代码

> 原文链接：<https://kylemayes.github.io/vulkanalia/setup/base_code.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码：**[main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/00_base_code.rs)

在开发环境一章中，我们创建了一个 Cargo 项目添加了必要的依赖项目。在本章中，我们会用下面的代码替换 `src/main.rs` 中的内容：

```rust
#![allow(
    dead_code,
    unused_variables,
    clippy::too_many_arguments,
    clippy::unnecessary_wraps
)]

use anyhow::Result;
use winit::dpi::LogicalSize;
use winit::event::{Event, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::{Window, WindowBuilder};

fn main() -> Result<()> {
    pretty_env_logger::init();

    // 创建窗口

    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
        .with_title("Vulkan Tutorial (Rust)")
        .with_inner_size(LogicalSize::new(1024, 768))
        .build(&event_loop)?;

    // 初始化应用程序

    let mut app = unsafe { App::create(&window)? };
    let mut destroying = false;
    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Poll;
        match event {
            // Render a frame if our Vulkan app is not being destroyed.
            Event::MainEventsCleared if !destroying =>
                unsafe { app.render(&window) }.unwrap(),
            // Destroy our Vulkan app.
            Event::WindowEvent { event: WindowEvent::CloseRequested, .. } => {
                destroying = true;
                *control_flow = ControlFlow::Exit;
                unsafe { app.destroy(); }
            }
            _ => {}
        }
    });
}

/// 我们的 Vulkan 应用程序
#[derive(Clone, Debug)]
struct App {}

impl App {
    /// 创建 Vulkan App
    unsafe fn create(window: &Window) -> Result<Self> {
        Ok(Self {})
    }

    /// 渲染帧
    unsafe fn render(&mut self, window: &Window) -> Result<()> {
        Ok(())
    }

    /// 销毁 Vulkan App
    unsafe fn destroy(&mut self) {}
}

/// 我们的 Vulkan 应用程序所使用的 Vulkan 句柄和相关属性
#[derive(Clone, Debug, Default)]
struct AppData {}
```

首先我们导入 `anyhow::Result`，这样我们就可以为程序中所有可失败的函数使用 `anyhow` 提供的 [`Result`](https://docs.rs/anyhow/latest/anyhow/type.Result.html) 类型。接下来我们导入所有 `winit` 类型，这些类型将被用于创建窗口并且启动窗口的事件循环。

接着是我们的 `main` 函数（它返回一个 `anyhow::Result`）。这个函数首先初始化 `pretty_env_logger`，它将会把日志打印到控制台（稍后会展示）。

接着，我们创建一个事件循环（event loop），并用 `winit` 和 `LogicalSize` 来创建一个窗口作为渲染的目标。`LogicalSize` 会根据你的显示器的 DPI 来缩放窗口。如果你想了解更多关于 UI 缩放的内容，可以阅读 [`winit` 文档](https://docs.rs/winit/latest/winit/dpi/index.html)。

接着，我们创建一个我们的 Vulkan 应用（`App`）的实例，并进入渲染循环。这个循环会持续将我们的场景渲染到窗口，直到你请求关闭窗口 —— 此时应用程序会被销毁并且程序会退出。`destroying` 标志是必要的，因为在应用程序被销毁时，我们不希望继续渲染场景，否则程序很可能会在尝试访问已被销毁的 Vulkan 资源时崩溃。

最后是 `App` 和 `AppData`，`App` 会被用来实现 Vulkan 程序所需的设置、渲染和析构逻辑。在接下来的章节中，我们都会围绕这个 Vulkan 程序工作。我们会创建非常多的 Vulkan 资源，`AppData` 会被用作容纳这些资源的容器，这样我们就可以轻松地将这些资源传递给函数。

这样做非常方便，因为我们接下来的很多章节都是加入一个接受 `&mut AppData` 的函数，创建并初始化 Vulkan 资源。这些函数会在 `App::create` 构造器中被调用来初始化我们的 Vulkan 应用。并且，在程序结束前，这些 Vulkan 资源会被 `App::destory` 方法销毁。

## 一个关于安全性的注解

所有的 Vulkan 函数，无论是原始函数还是它们的封装，在 `vulkanalia` 中都是被标记为 `unsafe` 的。这是因为许多 Vulkan 函数对如何调用它们作了限制，而 Rust 无法确保这些限制（除非引入一个更高阶的接口来隐藏 Vulkan API，例如 [`vulkano`](https://vulkano.rs)）。

本教程通过把所有调用 Vulkan 函数的函数和方法标记为 `unsafe` 来解决这一问题。这可以最大程度上减少语法噪音，但一个真实的程序中你可能希望自己封装一个安全的接口，并自行确保调用 Vulkan 函数时的正确性。

## 资源管理

正如在 C 中使用 `malloc` 分配的每一块内存都需要对应一个 `free` 调用一样，对于我们创建的每一个 Vulkan 对象，我们都要在不再需要它时显式地将其销毁。在 Rust 中，使用[资源分配即初始化（Resource Acuisition Is Initialization, RAII）](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)配合像 `Rc` 和 `Arc` 这样的智能指针来自动管理资源是可行的。然而，<https://vulkan-tutorial.com> 的作者选择在教程中显式地创建和销毁 Vulkan 对象，并且我也打算采用同样的方式。毕竟，Vulkan 的特点就是每一个操作都要显式地进行以避免错误，所以显式地管理对象的生命周期也是很好的学习 API 工作方式的途径。

在完成本教程之后，你就可以编写 Rust 结构体来包装 Vulkan 对象，并在其 `Drop` 实现中释放 Vulkan 对象，从而实现自动资源管理。对于大规模的 Vulkan 程序而言，RAII 是推荐的模式，但是为了学习，了解背后发生了什么也是很有好处的。

Vulkan 对象要么是由 `create_xxx` 之类的函数直接创建出来的，要么是通过另一个对象和 `allocate_xxx` 之类的函数分配出来的。在确保一个对象不会再使用之后，你需要用对应的 `destroy_xxx` 或者 `free_xxx` 来销毁它。这些函数的参数通常因对象的类型不同而有所不同，但是它们都有一个共同的参数：`allocator`。这是一个可选参数，允许你指定一个自定义内存分配器的回调函数。在本教程中我们会忽略这个参数，总是传递 `None`。
