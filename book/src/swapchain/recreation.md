# 重建交换链

> 原文链接：<https://kylemayes.github.io/vulkanalia/swapchain/recreation.html>
>
> Commit Hash: 72b9244ea1d53fa0cf40ce9dbf854c43286bf745

**本章代码:** [main.rs](https://github.com/chuigda/Vulkan-Tutorial-Rust-CN/tree/master/src/16_swapchain_recreation.rs)

现在应用程序能画出三角形了，但它还没有正确处理一些情况。窗口表面可能会发生变化，使得交换链不再与之兼容。导致这种情况发生的原因之一是窗口的大小发生了变化。我们必须捕获这些事件并重新创建交换链。

## 重建交换链

Create a new `App::recreate_swapchain` method that calls `create_swapchain` and all of the creation functions for the objects that depend on the swapchain or the window size.

创建一个新的 `App::recreate_swapchain` 方法，它调用 `create_swapchain` 来创建交换链，并调用所有依赖于交换链或者窗口大小的对象的创建函数：

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    self.device.device_wait_idle()?;
    create_swapchain(window, &self.instance, &self.device, &mut self.data)?;
    create_swapchain_image_views(&self.device, &mut self.data)?;
    create_render_pass(&self.instance, &self.device, &mut self.data)?;
    create_pipeline(&self.device, &mut self.data)?;
    create_framebuffers(&self.device, &mut self.data)?;
    create_command_buffers(&self.device, &mut self.data)?;
    self.data
        .images_in_flight
        .resize(self.data.swapchain_images.len(), vk::Fence::null());
    Ok(())
}
```

和上一章一样，我们首先调用 `device_wait_idle`，因为我们不能在资源正在被使用时修改它们。显然，我们要做的第一件事就是重建交换链本身。因为图像视图直接基于交换链图像，所以图像视图也需要被重建。因为渲染流程依赖于交换链图像的格式，所以渲染流程也需要被重建。在像窗口大小调整这样的操作中，交换链图像的格式改变的可能性很小，但是我们仍然需要处理这种情况。视口和裁剪矩形的大小在图形管线创建时指定，所以图形管线也需要被重建。使用动态状态来指定视口和裁剪矩形可以避免这种情况。然后，帧缓冲和指令缓冲也直接依赖于交换链图像。最后，我们调整了交换链图像的信号量列表的大小，因为重建后交换链图像的数量可能会发生变化。

为了确保在重建这些对象之前清理旧对象，我们应该将清理这些对象的代码从 `App::destroy` 方法中提取出来，移动到一个单独的 `App::destroy_swapchain` 方法中

```rust,noplaypen
unsafe fn recreate_swapchain(&mut self, window: &Window) -> Result<()> {
    self.device.device_wait_idle()?;
    self.destroy_swapchain();
    // ...
}

unsafe fn destroy_swapchain(&mut self) {

}
```

然后我们将所有在交换链刷新时重建的对象的清理代码从 `App::destroy` 移动到 `App::destroy_swapchain` 中：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.destroy_swapchain();

    self.data.in_flight_fences
        .iter()
        .for_each(|f| self.device.destroy_fence(*f, None));
    self.data.render_finished_semaphores
        .iter()
        .for_each(|s| self.device.destroy_semaphore(*s, None));
    self.data.image_available_semaphores
        .iter()
        .for_each(|s| self.device.destroy_semaphore(*s, None));
    self.device.destroy_command_pool(self.data.command_pool, None);
    self.device.destroy_device(None);
    self.instance.destroy_surface_khr(self.data.surface, None);

    if VALIDATION_ENABLED {
        self.instance.destroy_debug_utils_messenger_ext(self.data.messenger, None);
    }

    self.instance.destroy_instance(None);
}

unsafe fn destroy_swapchain(&mut self) {
    self.data.framebuffers
        .iter()
        .for_each(|f| self.device.destroy_framebuffer(*f, None));
    self.device.free_command_buffers(self.data.command_pool, &self.data.command_buffers);
    self.device.destroy_pipeline(self.data.pipeline, None);
    self.device.destroy_pipeline_layout(self.data.pipeline_layout, None);
    self.device.destroy_render_pass(self.data.render_pass, None);
    self.data.swapchain_image_views
        .iter()
        .for_each(|v| self.device.destroy_image_view(*v, None));
    self.device.destroy_swapchain_khr(self.data.swapchain, None);
}
```

我们也可以从头开始重建指令池，不过那样就太浪费了。因此我选择使用 `free_command_buffers` 函数清理现有的指令缓冲。这样我们就可以重用现有的指令池来分配新的指令缓冲。

这就是重建交换链所需的所有操作！然而，这样做的缺陷就是我们需要在创建新的交换链之前停止所有渲染操作。在旧交换链的图像上的绘制命令仍在执行时创建新的交换链是可能的。你需要将旧交换链传递给 `vk::SwapchainCreateInfoKHR` 结构体中的 `old_swapchain` 字段，并在使用完旧交换链后立即销毁它。

## 检测次优或过时的交换链

现在我们只需要确定什么时候必须重建交换链，并且调用 `App::recreate_swapchain` 方法。幸运地是，Vulkan 通常会在交换链不再适用时告诉我们。`acquire_next_image_khr` 和 `queue_present_khr` 命令可以返回以下特殊值来指示这一点。

* `vk::ErrorCode::OUT_OF_DATE_KHR` &ndash; 交换链与表面不再兼容，不能再用于渲染。通常发生在窗口大小调整之后。
* `vk::SuccessCode::SUBOPTIMAL_KHR` &ndash; 交换链仍然能向表面呈现内容，但是表面的属性不再与交换链完全匹配。

```rust,noplaypen
let result = self.device.acquire_next_image_khr(
    self.data.swapchain,
    u64::max_value(),
    self.data.image_available_semaphores[self.frame],
    vk::Fence::null(),
);

let image_index = match result {
    Ok((image_index, _)) => image_index as usize,
    Err(vk::ErrorCode::OUT_OF_DATE_KHR) => return self.recreate_swapchain(window),
    Err(e) => return Err(anyhow!(e)),
};
```

如果在尝试从交换链获取图像时发现交换链已经过时，那么就不能再向它呈现内容了。因此我们应该立即重建交换链，并在下一次 `App::render` 调用时再次尝试。

你也可以选择在交换链不再最优时重建交换链，但我选择在这种情况下继续进行渲染，因为我们已经获取了一个图像。因为 `vk::SuccessCode::SUBOPTIMAL_KHR` 被认为是一个成功的代码而不是一个错误代码，所以它将被 `match` 块中的 `Ok` 分支处理。

```rust,noplaypen
let result = self.device.queue_present_khr(self.data.present_queue, &present_info);

let changed = result == Ok(vk::SuccessCode::SUBOPTIMAL_KHR)
    || result == Err(vk::ErrorCode::OUT_OF_DATE_KHR);

if changed {
    self.recreate_swapchain(window)?;
} else if let Err(e) = result {
    return Err(anyhow!(e));
}
```

`queue_present_khr` 函数返回和 `acquire_next_image_khr` 相同的值，意义也相同。在这种情况下，如果交换链是次优的，我们也会重建交换链，因为我们想要最好的结果。

## 显式地处理窗口大小变化

尽管许多平台和驱动程序都会在窗口大小改变后自动触发 `vk::ErrorCode::OUT_OF_DATE_KHR`，但并不保证如此。这就是为什么我们要添加一些额外的代码来显式地处理窗口大小变化。首先在 `App` 结构体中添加一个新字段来追踪窗口大小是否发生了改变：

```rust,noplaypen
struct App {
    // ...
    resized: bool,
}
```

记得在 `App::create` 中将这个新字段初始化为 `false`。然后在 `App::render` 方法中，在调用 `queue_present_khr` 之后也检查这个标志：

```rust,noplaypen
let result = self.device.queue_present_khr(self.data.present_queue, &present_info);

let changed = result == Ok(vk::SuccessCode::SUBOPTIMAL_KHR)
    || result == Err(vk::ErrorCode::OUT_OF_DATE_KHR);

if self.resized || changed {
    self.resized = false;
    self.recreate_swapchain(window)?;
} else if let Err(e) = result {
    return Err(anyhow!(e));
}
```

注意要在 `queue_present_khr` 之后执行这个操作，以确保信号量处于一致的状态，否则一个已经发出信号的信号量可能永远不会被正确地等待。现在我们可以在 `main` 中的窗口事件 `match` 块中添加一个分支来实际检测调整大小：

```rust,noplaypen
match event {
    // ...
    Event::WindowEvent { event: WindowEvent::Resized(_), .. } => app.resized = true,
    // ...
}
```

现在尝试运行程序并调整窗口大小，看看帧缓冲是否确实与窗口一起正确调整大小。

## 处理窗口最小化

还有一种特殊的情况会导致交换链过时，那就是一种特殊的窗口调整大小：窗口最小化。这种情况很特殊，因为它会导致帧缓冲大小为 `0`。在本教程中，我们将通过在窗口最小化时不渲染帧来处理这种情况：

```rust,noplaypen
let mut app = unsafe { App::create(&window)? };
let mut destroying = false;
let mut minimized = false;
event_loop.run(move |event, _, control_flow| {
    *control_flow = ControlFlow::Poll;
    match event {
        Event::MainEventsCleared if !destroying && !minimized =>
            unsafe { app.render(&window) }.unwrap(),
        Event::WindowEvent { event: WindowEvent::Resized(size), .. } => {
            if size.width == 0 || size.height == 0 {
                minimized = true;
            } else {
                minimized = false;
                app.resized = true;
            }
        }
        // ...
    }
});
```

恭喜你，你已经完成了你的第一个行为良好的 Vulkan 程序！在下一章中，我们会避免在顶点着色器中硬编码顶点，并实际地使用一个顶点缓冲。
