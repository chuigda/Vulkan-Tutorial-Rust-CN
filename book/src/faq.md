# FAQ

> 原文链接：<https://kylemayes.github.io/vulkanalia/faq.html>
>
> Commit Hash: 7becee96b0029bf721f833039c00ea2a417714dd

本页面列举了在开发 Vulkan 应用时可能遇到的常见问题及其解决方案。

* **（MacOS）** 我安装了 Vulkan SDK，但我运行 Vulkan 应用程序的时候遇到了找不到 `libvulkan.dylib` 的错误 —— 参见[MacOS Vulkan SDK 安装说明中的`环境配置`章节](./development_environment.md#setup-environment)。

* **我在核心校验层中遇到了访问冲突（access violation）错误** &ndash; 确保未运行 MSI Afterburner / RivaTuner Statistics Server，因为它们和 Vulkan 之间存在一些兼容性问题。

* **我看不到任何来自校验层的消息/校验层不可用** &ndash; 首先确保校验层有机会打印错误信息，请在程序退出后保持终端窗口打开。在 Visual Studio 中，你可以通过使用 Ctrl-F5 而不是 F5 来运行程序；在 Linux 中，你可以从终端窗口执行程序。如果仍然没有消息，并且你确信校验层已启用，那么你应该按照[此页面上的“Verify the Installation”说明](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)来确保 Vulkan SDK 已正确安装。同时确保你的 SDK 版本至少为 1.1.106.0，以支持 `VK_LAYER_KHRONOS_validation` 校验层。

* **`vkCreateSwapchainKHR` 在 `SteamOverlayVulkanLayer64.dll` 中引发错误** &ndash; 这似乎是测试版 Steam 客户端中的一个兼容性问题。以下有几个也许可行的解决方法：
  * 退出 Steam 测试计划
  * 将环境变量 `DISABLE_VK_LAYER_VALVE_steam_overlay_1` 设置为`1`。
  * 删除注册表中 `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` 下的 Steam overlay Vulkan layer 项目。

示例:

![](./images/steam_layers_env.png)
