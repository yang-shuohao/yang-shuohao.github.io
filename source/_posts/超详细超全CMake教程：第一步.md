---
title: 超详细超全CMake教程：第一步：一个基本的起点
date: 2021-09-01 22:59:55
tags: CMake
categories: CMake
cover: https://img-blog.csdnimg.cn/73d242c110714789b7277f02cb9fdd9c.png
---

# 第一步：一个基本的起点
最基本的项目是从源代码文件构建的可执行文件。对于简单的项目，只需要一个三行CMakeLists.txt文件。这将是我们教程的起点。在Step1目录中创建一个 CMakeLists.txt文件，添加如下代码：

```cpp
cmake_minimum_required(VERSION 3.10)

# set the project name
project(Tutorial)

# add the executable
add_executable(Tutorial tutorial.cxx)
```

请注意，此示例在CMakeLists.txt文件中使用小写命令。CMake 支持大写、小写和大小写混合命令。目录中提供了tutorial.cxx的源代码，Step1可用于计算数字的平方根。
## 添加版本号和配置的头文件
我们将添加的第一个功能是为我们的可执行文件和项目提供版本号。虽然我们可以只在源代码中做到这一点，但使用 CMakeLists.txt提供了更大的灵活性。

首先，修改CMakeLists.txt文件以使用project() 命令来设置项目名称和版本号。

```cpp
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)
```
然后，配置一个头文件将版本号传递给源代码：

```cpp
configure_file(TutorialConfig.h.in TutorialConfig.h)
```
由于配置的文件将写入二叉树，我们必须将该目录添加到路径列表中以搜索包含文件。将以下行添加到CMakeLists.txt文件末尾：

```cpp
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```
使用您喜欢的编辑器，在Step1目录下创建TutorialConfig.h.in文件输入以下内容：

```cpp
// the configured options and settings for Tutorial
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```
当 CMake 配置这个头文件时，@Tutorial_VERSION_MAJOR@和@Tutorial_VERSION_MINOR@的值将被替换。

接下来修改tutorial.cxx添加头文件 TutorialConfig.h.

最后，让我们通过更新打印出可执行文件名称和版本号， tutorial.cxx如下所示：

```cpp
if (argc < 2) {
    // report version
    std::cout << argv[0] << " Version " << Tutorial_VERSION_MAJOR << "."
              << Tutorial_VERSION_MINOR << std::endl;
    std::cout << "Usage: " << argv[0] << " number" << std::endl;
    return 1;
  }
```
## 指定 C++ 标准
接下来让我们通过替换tutorial.cxx文件里面的atof为 std::stodin，为我们的项目添加一些 C++11 特性。同时，删除 .#include <cstdlib>

```cpp
 const double inputValue = std::stod(argv[1]);
```
我们需要在 CMake 代码中明确声明它应该使用正确的标志。在 CMake 中启用对特定 C++ 标准的支持的最简单方法是使用CMAKE_CXX_STANDARD变量。对于本教程，设置CMakeLists.txt文件中的变量CMAKE_CXX_STANDARD值为11，CMAKE_CXX_STANDARD_REQUIRED 值为True。确保CMAKE_CXX_STANDARD在调用上方添加声明add_executable。

```cpp
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```
CMakeLists.txt完整代码：

```cpp
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Tutorial VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# add the executable
add_executable(Tutorial tutorial.cxx)

configure_file(TutorialConfig.h.in TutorialConfig.h)

target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```
tutorial.cxx完整代码：

```cpp
// A simple program that computes the square root of a number
#include <cmath>
#include <iostream>
#include <string>
#include "TutorialConfig.h"

int main(int argc, char* argv[])
{
 if (argc < 2) {
    // report version
    std::cout << argv[0] << " Version " << Tutorial_VERSION_MAJOR << "."
              << Tutorial_VERSION_MINOR << std::endl;
    std::cout << "Usage: " << argv[0] << " number" << std::endl;
    return 1;
  }

  // convert input to double
  const double inputValue = std::stod(argv[1]);

  // calculate square root
  const double outputValue = sqrt(inputValue);
  std::cout << "The square root of " << inputValue << " is " << outputValue
            << std::endl;
  return 0;
}

```
TutorialConfig.h.in完整代码：

```cpp
// the configured options and settings for Tutorial
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

## 构建和测试
运行cmake 可执行文件或 cmake-gui 配置项目，然后使用您选择的构建工具构建它。

例如，我们可以从命令行导航到 Help/guide/tutorial目录并创建一个构建文件夹：

```cpp
mkdir Step1_build
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/721d3be2bf264042bd8973e3838527f0.png)

接下来，导航到构建目录并运行 CMake 以配置项目并生成本机构建系统：

```cpp
cd Step1_build
cmake ../Step1
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/73d242c110714789b7277f02cb9fdd9c.png)

然后调用该构建系统来实际编译/链接项目：

```cpp
cmake --build .
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/5b233debe95f48bbbf7c60ac1da08849.png)    

构建成功以后的项目：   

![在这里插入图片描述](https://img-blog.csdnimg.cn/2666c3cc25974fc2b7fa8b9eeda477d8.png)   

恭喜你完成了第一步。