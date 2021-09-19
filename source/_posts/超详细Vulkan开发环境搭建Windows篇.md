---
title: 超详细Vulkan开发环境搭建Windows篇
tags: 开发环境搭建
categories: Vulkan
cover: 'https://img-blog.csdnimg.cn/7fe287dcb72c49cfbf852209d12d17b2.gif#pic_center'
abbrlink: 61b90d80
date: 2021-09-11 09:55:27
---

# 1.Vulkan SDK
开发 Vulkan 应用程序所需的最重要组件是 SDK。它包括头文件、标准验证层、调试工具和 Vulkan 函数的加载程序。加载程序在运行时查找驱动程序中的函数，类似于 OpenGL 的 GLEW - 如果您熟悉它。
可以 使用页面底部的按钮从[LunarG](https://vulkan.lunarg.com/) 网站下载 SDK 。您不必创建帐户，但它可以让您访问一些可能对您有用的其他文档。    

![在这里插入图片描述](https://img-blog.csdnimg.cn/df92c24d6acf454cb21ff9ed55110192.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/1eef8f962e1d40b4ad06f90ce8b67ec1.png)   

继续安装，注意SDK的安装位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/19ab74435db44b24ad6812e3fb491fbd.png)    

我们要做的第一件事是验证您的显卡和驱动程序是否正确支持 Vulkan。进入SDK安装目录，打开Bin目录，运行vkcube.exedemo。您应该看到以下内容：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7fe287dcb72c49cfbf852209d12d17b2.gif#pic_center)

如果您收到错误消息，请确保您的驱动程序是最新的，包括 Vulkan 运行时并且您的显卡受支持。

# glm
[glm下载地址](https://github.com/g-truc/glm/releases)
![在这里插入图片描述](https://img-blog.csdnimg.cn/6212207b028d411280564ae68d412d8b.png)

# GLFW
[GLFW下载地址](https://www.glfw.org/download.html)
![在这里插入图片描述](https://img-blog.csdnimg.cn/b2c70882565f4bd1a9fa29af9edb7e05.png)

# 创建项目
打开VS2019，新建一个C++空项目，然后添加一个Main.cpp文件。

## 配置项目属性
右键创建的项目，选择最下面的属性，打开属性面板。
选择C/C++下面的General，在第一个添加包含目录里面添加Vulkan包含目录，glfw包含目录和glm包含目录。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c69d9998fc4d431ea9f474382370c3cb.png)

选择Linker下面的General，在添加库目录里面添加Vulkan和glfw的库目录。

![在这里插入图片描述](https://img-blog.csdnimg.cn/635eceeda4444a48ae5ab9b152f30c91.png)

选择Linker下面的Input，在添加依赖里面输入vulkan-1.lib和glfw3.lib。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8353662194474378980dabb0e00ee2f9.png)

# 测试
到这里Vulkan的开发环境就搭建好了，在创建的Main.cpp里面输入下面代码进行测试。

```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```
出现白色窗口就说明成功了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7e0f179f7a5441f593a6e998c5279303.png)

# 结尾
如果你遇到什么问题可以在评论区告诉我哦。



