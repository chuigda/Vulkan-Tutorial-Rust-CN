# FAQ

本页面列举了在开发Vulkan应用时可能遇到的常见问题的解决方案。

* **我在核心校验层中遇到了访问冲突错误** &ndash; 确保未运行MSI Afterburner / RivaTuner Statistics Server,因为它与Vulkan存在一些兼容性问题。

* **我看不到任何来自校验层的消息/校验层不可用** &ndash; 首先要确保校验层有机会打印错误信息，请在程序退出后保持终端窗口打开。在Visual Studio中，你可以通过使用Ctrl—F5而不是F5来运行程序，在Linux中，可以通过从终端窗口执行程序来实现。如果仍然没有消息并且你确信校验层已启用，那么你应该按照[此页面上的“Verify the Installation"说明](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)来确保您的VulkanSDK已正确安装。同时确保你的SDK版本至少为1.1.106.0，以支持`VK_LAYER_KHRONOS_validation`层。

* **vkCreateSwapchainKHR在SteamOverlayVulkanLayer64.dll中触发错误** &ndash; 这似乎是Steam客户端测试版中的一个兼容性问题。以下有几个可能的解决方法：
  * 退出Steam测试版计划。
  * 将环境变量`DISABLE_VK_LAYER_VALVE_steam_overlay_1`设置为1。
  * 删除注册表中`HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers`下的 Steam overlay Vulkan layer项目。

例子:

![](./images/steam_layers_env.png)
