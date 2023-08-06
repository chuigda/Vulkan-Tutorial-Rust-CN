# 渲染与呈现

> 原文链接：<https://kylemayes.github.io/vulkanalia/drawing/rendering_and_presentation.html>
> 
> Commit Hash: f083d3b38f8be37555a1126cd90f6b73c8679d99

**本章代码:** [main.rs](https://github.com/KyleMayes/vulkanalia/tree/master/tutorial/src/15_hello_triangle.rs)

在本章中我们会把所有东西组合起来。我们将实现 `App::render` 函数，该函数会由主循环调用，将三角形渲染到屏幕上。

## 同步

`App::render` 函数会进行以下操作：

* 从交换链获取一张图像
* 使用该图像作为帧缓冲的附件，执行指令缓冲
* 将该图像返还到交换链，以供呈现

每个操作都是通过单个函数调用来启动的，但它们会异步地（asynchronously）执行。函数调用会在操作实际完成之前返回，且操作之间的执行顺序也是不确定的。然而很不幸的是，我们的每一步操作实际上都依赖于前一步操作的完成。

有两种方法可以用于同步交换链事件：信号量（semaphores）和栅栏（fences）。它们都是可以用于协调操作的对象，这是通过让一个操作发出信号、另一个操作等待信号量或栅栏的状态从未发出信号（unsignaled）变为已发出信号（signaled）来实现的。

区别在于，栅栏的状态可以通过 `wait_for_fences` 等函数从程序中访问，而信号量则不行。栅栏主要用于同步应用程序本身与渲染操作，而信号量则用于同步指令队列内或跨队列的操作。我们希望同步绘制指令和呈现操作，因此信号量是最佳选择。

## 信号量

我们需要两个信号量，一个用于传递图像已被获取并可以用于渲染的信号，另一个用于传递渲染已完成并可以进行呈现的信号。在 `AppData` 中创建两个字段来存储这些信号量对象：

```rust,noplaypen
struct AppData {
    // ...
    image_available_semaphore: vk::Semaphore,
    render_finished_semaphore: vk::Semaphore,
}
```

添加一个 `create_sync_objects` 函数来创建信号量：

```rust,noplaypen
impl App {
    unsafe fn create(window: &Window) -> Result<Self> {
        // ...
        create_command_buffers(&device, &mut data)?;
        create_sync_objects(&device, &mut data)?;
        // ...
    }
}

unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    Ok(())
}
```

创建信号量需要填充 `vk::SemaphoreCreateInfo` 结构体，不过在当前版本的 API 中它并没有任何必填字段。

```rust,noplaypen
unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    let semaphore_info = vk::SemaphoreCreateInfo::builder();

    Ok(())
}
```

未来的 Vulkan API 或者扩展可能会为 `flags` 和 `p_next` 参数添加功能，就像它为其他结构体所做的那样。信号量的创建也是熟悉的模式：

```rust,noplaypen
data.image_available_semaphore = device.create_semaphore(&semaphore_info, None)?;
data.render_finished_semaphore = device.create_semaphore(&semaphore_info, None)?;
```

信号量应该在程序结束时清理，这时候所有指令都已经完成，用不着再进行同步了：

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.device.destroy_semaphore(self.data.render_finished_semaphore, None);
    self.device.destroy_semaphore(self.data.image_available_semaphore, None);
    // ...
}
```

## 从交换链获取图像

正如我们之前所提到的，在 `App::render` 函数中要做的第一件事就死活从交换链取得一张图像。回想一下，交换链是一个扩展特性，因此我们必须使用一个带有 `*_khr` 后缀的函数：

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    let image_index = self
        .device
        .acquire_next_image_khr(
            self.data.swapchain,
            u64::max_value(),
            self.data.image_available_semaphore,
            vk::Fence::null(),
        )?
        .0 as usize;

        let wait_semaphores = &[self.data.image_available_semaphore];
let wait_stages = &[vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
let command_buffers = &[self.data.command_buffers[image_index as usize]];
let signal_semaphores = &[self.data.render_finished_semaphore];
let submit_info = vk::SubmitInfo::builder()
    .wait_semaphores(wait_semaphores)
    .wait_dst_stage_mask(wait_stages)
    .command_buffers(command_buffers)
    .signal_semaphores(signal_semaphores);

    self.device.queue_submit(
    self.data.graphics_queue, &[submit_info], vk::Fence::null())?;

    Ok(())
}
```

`acquire_next_image_khr` 的第一个参数是我们将要从中获取图像的交换链。第二个参数指定了一个以纳秒为单位的超时时间，使用 64 位无符号整数的最大值可以禁用超时。

下一个参数指定了在呈现引擎使用完图像后要发出信号的同步对象，可以是信号量或者栅栏，也可以两者都指定。我们将在这里使用 `image_available_semaphore`，它发出信号的时刻就是我们可以开始绘制的时候。

这个函数返回将会被获取的交换链图像的索引。这个索引指向 `swapchain_images` 数组中的 `vk::Image`。我们将使用这个索引来选择正确的命令缓冲。

## 提交指令缓冲

队列的提交和同步是通过 `vk::SubmitInfo` 结构体来配置的：

```rust,noplaypen
let wait_semaphores = &[self.data.image_available_semaphore];
let wait_stages = &[vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
let command_buffers = &[self.data.command_buffers[image_index as usize]];
let signal_semaphores = &[self.data.render_finished_semaphore];
let submit_info = vk::SubmitInfo::builder()
    .wait_semaphores(wait_semaphores)
    .wait_dst_stage_mask(wait_stages)
    .command_buffers(command_buffers)
    .signal_semaphores(signal_semaphores);
```

前两个参数 `wait_semaphores` 和 `wait_dst_stage_mask` 指定在指令缓冲执行前要等待的信号量，以及在管线的哪个阶段等待。我们希望在图像可用之前不要写入颜色，因此我们指定了写入颜色附件的管线阶段。这意味着理论上来说实现可以进行若干优化，例如可以在图像还不可用的时候就开始执行我们的顶点着色器。`wait_stages` 数组中的每个条目都对应于 `wait_semaphores` 中相同索引的信号量。

下一个参数，`command_buffers`，指定了要提交执行的指令缓冲。正如之前提到的，我们应该提交绑定了我们刚获取的交换链图像的指令缓冲。

最后一个参数 `signal_semaphores` 指定了在指令缓冲执行完毕后要发出信号的信号量。在我们的例子中，我们使用 `render_finished_semaphore`。

```rust,noplaypen
self.device.queue_submit(
    self.data.graphics_queue, &[submit_info], vk::Fence::null())?;
```

现在我们可以用 `queue_submit` 来将指令缓冲提交到图形队列了。这个函数接受一个 `vk::SubmitInfo` 结构体的数组作为参数，这样做是为了在工作量很大的时候提高效率。最后一个参数引用了一个可选的栅栏，当指令缓冲执行完毕时会发出信号。我们已经在用信号量来进行同步了，因此我们只传递一个 `vk::Fence::null()`。

## 子流程依赖

还记得渲染流程中自动进行图像布局转换的子流程吗？这些转换是由*子流程依赖*（subpass dependencies）控制的，它们指定了子流程之间的内存和执行依赖关系。我们现在只有一个子流程，但是在这个子流程之前和之后的操作也被视为隐式的“子流程”。

有两个内建的子流程依赖能在渲染流程开始前和结束后进行布局转换，但前者进行转换的时机并不正确 —— 它假设转换发生在管线开始的时候，但在那个时候我们还没有获取到图像！有两种方法可以解决这个问题。我们可以将 `image_available_semaphore` 的 `wait_stages` 改为 `vk::PipelineStageFlags::TOP_OF_PIPE`，以确保图像可用之前渲染流程不会开始。或者，我们可以让渲染流程等待 `vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT` 阶段。我决定在这里使用第二种方法，因为这是个了解子流程依赖的工作原理的好机会。

Subpass dependencies are specified in `vk::SubpassDependency` structs. Go to our `^create_render_pass` function and add one:

子流程依赖是由 `vk::SubpassDependency` 结构体来指定的。在 `create_render_pass` 函数中添加一个：

```rust,noplaypen
let dependency = vk::SubpassDependency::builder()
    .src_subpass(vk::SUBPASS_EXTERNAL)
    .dst_subpass(0)
    // continued...
```

前两个字段 `src_subpass` 和 `dst_subpass` 指定了依赖和被依赖的子流程的索引。特殊值 `vk::SUBPASS_EXTERNAL` 指的是隐式子流程，它位于渲染流程开始前或结束后，具体的语义取决于它是在 `src_subpass` 还是 `dst_subpass` 中指定的。索引 `0` 指的是我们的子流程，也是唯一的子流程。`dst_subpass` 必须始终大于 `src_subpass`，以防止依赖图中出现循环（除非其中一个子流程是 `vk::SUBPASS_EXTERNAL`）。

```rust,noplaypen
    .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
    .src_access_mask(vk::AccessFlags::empty())
```

接下来的两个字段制订了要等待的操作，以及这些操作会在哪个阶段发生。我们需要等待交换链完成对图像的读取，然后才能访问它。这可以通过等待颜色附件输出阶段本身来实现。

```rust,noplaypen
    .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
    .dst_access_mask(vk::AccessFlags::COLOR_ATTACHMENT_WRITE);
```

The operations that should wait on this are in the color attachment stage and involve the writing of the color attachment. These settings will prevent the transition from happening until it's actually necessary (and allowed): when we want to start writing colors to it.

```rust,noplaypen
let attachments = &[color_attachment];
let subpasses = &[subpass];
let dependencies = &[dependency];
let info = vk::RenderPassCreateInfo::builder()
    .attachments(attachments)
    .subpasses(subpasses)
    .dependencies(dependencies);
```

The `vk::RenderPassCreateInfo` struct has a field to specify an array of dependencies.

## Presentation

The last step of drawing a frame is submitting the result back to the swapchain to have it eventually show up on the screen. Presentation is configured through a `vk::PresentInfoKHR` structure at the end of the `App::render` function.

```rust,noplaypen
let swapchains = &[self.data.swapchain];
let image_indices = &[image_index as u32];
let present_info = vk::PresentInfoKHR::builder()
    .wait_semaphores(signal_semaphores)
    .swapchains(swapchains)
    .image_indices(image_indices);
```

The first parameter specifies which semaphores to wait on before presentation can happen, just like `vk::SubmitInfo`.

The next two parameters specify the swapchains to present images to and the index of the image for each swapchain. This will almost always be a single one.

There is one last optional parameter called `results`. It allows you to specify an array of `vk::Result` values to check for every individual swapchain if presentation was successful. It's not necessary if you're only using a single swapchain, because you can simply use the return value of the present function.

```rust,noplaypen
self.device.queue_present_khr(self.data.present_queue, &present_info)?;
```

The `queue_present_khr` function submits the request to present an image to the swapchain. We'll modify the error handling for both `acquire_next_image_khr` and `queue_present_khr` in the next chapter, because their failure does not necessarily mean that the program should terminate, unlike the functions we've seen so far.

If you did everything correctly up to this point, then you should now see something resembling the following when you run your program:

![](../images/triangle.png)

>This colored triangle may look a bit different from the one you're used to seeing in graphics tutorials. That's because this tutorial lets the shader interpolate in linear color space and converts to sRGB color space afterwards. See [this blog post](https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9) for a discussion of the difference.

Yay! Unfortunately, you'll see that when validation layers are enabled, the program crashes as soon as you close it. The messages printed to the terminal from `debug_callback` tell us why:

![](../images/semaphore_in_use.png)

Remember that all of the operations in `App::render` are asynchronous. That means that when we call `App::destroy` before exiting the loop in `main`, drawing and presentation operations may still be going on. Cleaning up resources while that is happening is a bad idea.

To fix that problem, we should wait for the logical device to finish operations using `device_wait_idle` before calling `App::destroy`:

```rust,noplaypen
Event::WindowEvent { event: WindowEvent::CloseRequested, .. } => {
    destroying = true;
    *control_flow = ControlFlow::Exit;
    unsafe { app.device.device_wait_idle().unwrap(); }
    unsafe { app.destroy(); }
}
```

You can also wait for operations in a specific command queue to be finished with `queue_wait_idle`. These functions can be used as a very rudimentary way to perform synchronization. You'll see that the program no longer crashes when closing the window (though you will see some errors related to synchronization if you have the validation layers enabled).

## Frames in flight

If you run your application with validation layers enabled now you may either get errors or notice that the memory usage slowly grows. The reason for this is that the application is rapidly submitting work in the `App::render` function, but doesn't actually check if any of it finishes. If the CPU is submitting work faster than the GPU can keep up with then the queue will slowly fill up with work. Worse, even, is that we are reusing the `image_available_semaphore` and `render_finished_semaphore` semaphores, along with the command buffers, for multiple frames at the same time!

The easy way to solve this is to wait for work to finish right after submitting it, for example by using `queue_wait_idle` (note: don't actually make this change):

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    self.device.queue_present_khr(self.data.present_queue, &present_info)?;
    self.device.queue_wait_idle(self.data.present_queue)?;

    Ok(())
}
```

However, we are likely not optimally using the GPU in this way, because the whole graphics pipeline is only used for one frame at a time right now. The stages that the current frame has already progressed through are idle and could already be used for a next frame. We will now extend our application to allow for multiple frames to be *in-flight* while still bounding the amount of work that piles up.

Start by adding a constant at the top of the program that defines how many frames should be processed concurrently:

```rust,noplaypen
const MAX_FRAMES_IN_FLIGHT: usize = 2;
```

Each frame should have its own set of semaphores in `AppData`:

```rust,noplaypen
struct AppData {
    // ...
    image_available_semaphores: Vec<vk::Semaphore>,
    render_finished_semaphores: Vec<vk::Semaphore>,
}
```

The `create_sync_objects` function should be changed to create all of these:

```rust,noplaypen
unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    let semaphore_info = vk::SemaphoreCreateInfo::builder();

    for _ in 0..MAX_FRAMES_IN_FLIGHT {
        data.image_available_semaphores
            .push(device.create_semaphore(&semaphore_info, None)?);
        data.render_finished_semaphores
            .push(device.create_semaphore(&semaphore_info, None)?);
    }

    Ok(())
}
```

Similarly, they should also all be cleaned up:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.render_finished_semaphores
        .iter()
        .for_each(|s| self.device.destroy_semaphore(*s, None));
    self.data.image_available_semaphores
        .iter()
        .for_each(|s| self.device.destroy_semaphore(*s, None));
    // ...
}
```

To use the right pair of semaphores every time, we need to keep track of the current frame. We will use a frame index for that purpose which we'll add to `App` (initialize it to `0` in `App::create`):

```rust,noplaypen
struct App {
    // ...
    frame: usize,
}
```

The `App::render` function can now be modified to use the right objects:

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    let image_index = self
        .device
        .acquire_next_image_khr(
            self.data.swapchain,
            u64::max_value(),
            self.data.image_available_semaphores[self.frame],
            vk::Fence::null(),
        )?
        .0 as usize;

    // ...

    let wait_semaphores = &[self.data.image_available_semaphores[self.frame]];

    // ...

    let signal_semaphores = &[self.data.render_finished_semaphores[self.frame]];

    // ...

    Ok(())
}
```

Of course, we shouldn't forget to advance to the next frame every time:

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    self.frame = (self.frame + 1) % MAX_FRAMES_IN_FLIGHT;

    Ok(())
}
```

By using the modulo (%) operator, we ensure that the frame index loops around after every `MAX_FRAMES_IN_FLIGHT` enqueued frames.

Although we've now set up the required objects to facilitate processing of multiple frames simultaneously, we still don't actually prevent more than `MAX_FRAMES_IN_FLIGHT` from being submitted. Right now there is only GPU-GPU synchronization and no CPU-GPU synchronization going on to keep track of how the work is going. We may be using the frame #0 objects while frame #0 is still in-flight!

To perform CPU-GPU synchronization, Vulkan offers a second type of synchronization primitive called *fences*. Fences are similar to semaphores in the sense that they can be signaled and waited for, but this time we actually wait for them in our own code. We'll first create a fence for each frame in `AppData`:

```rust,noplaypen
struct AppData {
    // ...
    in_flight_fences: Vec<vk::Fence>,
}
```

We'll create the fences together with the semaphores in the `create_sync_objects` function:

```rust,noplaypen
unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    let semaphore_info = vk::SemaphoreCreateInfo::builder();
    let fence_info = vk::FenceCreateInfo::builder();

    for _ in 0..MAX_FRAMES_IN_FLIGHT {
        data.image_available_semaphores
            .push(device.create_semaphore(&semaphore_info, None)?);
        data.render_finished_semaphores
            .push(device.create_semaphore(&semaphore_info, None)?);

        data.in_flight_fences.push(device.create_fence(&fence_info, None)?);
    }

    Ok(())
}
```

The creation of fences (`vk::Fence`) is very similar to the creation of semaphores. Also make sure to clean up the fences in `App::destroy`:

```rust,noplaypen
unsafe fn destroy(&mut self) {
    self.data.in_flight_fences
        .iter()
        .for_each(|f| self.device.destroy_fence(*f, None));
    // ...
}
```

We will now change `App::render` to use the fences for synchronization. The `queue_submit` call includes an optional parameter to pass a fence that should be signaled when the command buffer finishes executing. We can use this to signal that a frame has finished.

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    self.device.queue_submit(
        self.data.graphics_queue,
        &[submit_info],
        self.data.in_flight_fences[self.frame],
    )?;

    // ...
}
```

Now the only thing remaining is to change the beginning of `App::render` to wait for the frame to be finished:

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    self.device.wait_for_fences(
        &[self.data.in_flight_fences[self.frame]],
        true,
        u64::max_value(),
    )?;

    self.device.reset_fences(&[self.data.in_flight_fences[self.frame]])?;

    // ...
}
```

The `wait_for_fences` function takes an array of fences and waits for either any or all of them to be signaled before returning. The `true` we pass here indicates that we want to wait for all fences, but in the case of a single one it obviously doesn't matter. Just like `acquire_next_image_khr` this function also takes a timeout. Unlike the semaphores, we manually need to restore the fence to the unsignaled state by resetting it with the `reset_fences` call.

If you run the program now, you'll notice something something strange. The application no longer seems to be rendering anything and might even be frozen.

That means that we're waiting for a fence that has not been submitted. The problem here is that, by default, fences are created in the unsignaled state. That means that `wait_for_fences` will wait forever if we haven't used the fence before. To solve that, we can change the fence creation to initialize it in the signaled state as if we had rendered an initial frame that finished:

```rust,noplaypen
unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    // ...

    let fence_info = vk::FenceCreateInfo::builder()
        .flags(vk::FenceCreateFlags::SIGNALED);

    // ...
}
```

The memory leak is gone now, but the program is not quite working correctly yet. If `MAX_FRAMES_IN_FLIGHT` is higher than the number of swapchain images or `acquire_next_image_khr` returns images out-of-order then it's possible that we may start rendering to a swapchain image that is already *in flight*. To avoid this, we need to track for each swapchain image if a frame in flight is currently using it. This mapping will refer to frames in flight by their fences so we'll immediately have a synchronization object to wait on before a new frame can use that image.

First add a new list called `images_in_flight` to `AppData` to track this:

```rust,noplaypen
struct AppData {
    // ...
    in_flight_fences: Vec<vk::Fence>,
    images_in_flight: Vec<vk::Fence>,
}
```

Prepare it in `create_sync_objects`:

```rust,noplaypen
unsafe fn create_sync_objects(device: &Device, data: &mut AppData) -> Result<()> {
    // ...

    data.images_in_flight = data.swapchain_images
        .iter()
        .map(|_| vk::Fence::null())
        .collect();

    Ok(())
}
```

Initially not a single frame is using an image so we explicitly initialize it to *no fence*. Now we'll modify `App::render` to wait on any previous frame that is using the image that we've just been assigned for the new frame:

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    let image_index = self
        .device
        .acquire_next_image_khr(
            self.data.swapchain,
            u64::max_value(),
            self.data.image_available_semaphores[self.frame],
            vk::Fence::null(),
        )?
        .0 as usize;

    if !self.data.images_in_flight[image_index as usize].is_null() {
        self.device.wait_for_fences(
            &[self.data.images_in_flight[image_index as usize]],
            true,
            u64::max_value(),
        )?;
    }

    self.data.images_in_flight[image_index as usize] =
        self.data.in_flight_fences[self.frame];

    // ...
}
```

Because we now have more calls to `wait_for_fences`, the `reset_fences` call should be **moved**. It's best to simply call it right before actually using the fence:

```rust,noplaypen
unsafe fn render(&mut self, window: &Window) -> Result<()> {
    // ...

    self.device.reset_fences(&[self.data.in_flight_fences[self.frame]])?;

    self.device.queue_submit(
        self.data.graphics_queue,
        &[submit_info],
        self.data.in_flight_fences[self.frame],
    )?;

    // ...
}
```

We've now implemented all the needed synchronization to ensure that there are no more than two frames of work enqueued and that these frames are not accidentally using the same image. Note that it is fine for other parts of the code, like the final cleanup, to rely on more rough synchronization like `device_wait_idle`. You should decide on which approach to use based on performance requirements.

To learn more about synchronization through examples, have a look at [this extensive overview](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present) by Khronos.

## Conclusion

A little over 600 (non-empty) lines of code later, we've finally gotten to the stage of seeing something pop up on the screen! Bootstrapping a Vulkan program is definitely a lot of work, but the take-away message is that Vulkan gives you an immense amount of control through its explicitness. I recommend you to take some time now to reread the code and build a mental model of the purpose of all of the Vulkan objects in the program and how they relate to each other. We'll be building on top of that knowledge to extend the functionality of the program from this point on.

In the next chapter we'll deal with one more small thing that is required for a well-behaved Vulkan program.
