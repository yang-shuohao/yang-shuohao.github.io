---
title: 超详细超全CMake教程：第二步添加库
tags: CMake
categories: CMake
cover: 'https://img-blog.csdnimg.cn/e8632c1478e146daa98a0fbb8367518b.png'
abbrlink: 1a358483
date: 2021-09-03 00:30:45
---


# 第 2 步：添加库
现在我们将向我们的项目添加一个库。这个库将包含我们自己的计算数字平方根的实现。然后可执行文件可以使用这个库代替编译器提供的标准平方根函数。
在本教程中，我们将把库放到MathFunctions里面。 此目录已包含头文件 MathFunctions.h和源文件mysqrt.cxx。源文件有一个被调用的函数mysqrt，它提供与编译器sqrt函数类似的功能。
在MathFunctions 目录中创建一个CMakeLists.txt文件并添加如下代码：

```cpp
add_library(MathFunctions mysqrt.cxx)
```
为了使用新库，我们将在顶级CMakeLists.txt文件中添加一个 add_subdirectory() ，以便构建库。我们将新库添加到可执行文件中，并添加MathFunctions为包含目录，以便mysqrt.h可以找到头文件。顶级CMakeLists.txt文件的最后几行现在应该如下所示：

```cpp
# add the MathFunctions library
add_subdirectory(MathFunctions)

# add the executable
add_executable(Tutorial tutorial.cxx)

target_link_libraries(Tutorial PUBLIC MathFunctions)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                          "${PROJECT_BINARY_DIR}"
                          "${PROJECT_SOURCE_DIR}/MathFunctions"
                          )
```
现在让我们将MathFunctions库设为可选。虽然对于本教程来说确实没有任何必要这样做，但对于较大的项目，这很常见。第一步是在顶级CMakeLists.txt文件中添加一个选项 。

```cpp
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file(TutorialConfig.h.in TutorialConfig.h)
```

此选项将显示在 cmake-gui 和 ccmake中，默认值为ON，用户可以更改此值。此设置将存储在缓存中，以便用户每次在构建目录上运行 CMake 时无需设置该值。
下一个更改是使MathFunctions库的构建和链接有判断条件。为此，我们将顶级CMakeLists.txt 文件的末尾更改为如下所示：

```cpp
if(USE_MYMATH)
  add_subdirectory(MathFunctions)
  list(APPEND EXTRA_LIBS MathFunctions)
  list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions")
endif()

# add the executable
add_executable(Tutorial tutorial.cxx)

target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS})

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           ${EXTRA_INCLUDES}
                           )
```

请注意使用EXTRA_LIBS该变量收集任何可选库，以便稍后链接到可执行文件中。EXTRA_INCLUDES该变量类似地用于可选的头文件。这是处理许多可选组件时的经典方法，我们将在下一步中介绍现代方法。
对源代码的相应更改相当简单。首先，在tutorial.cxx中包括MathFunctions.h头文件：

```cpp
#ifdef USE_MYMATH
#  include "MathFunctions.h"
#endif
```

然后，在tutorial.cxx中添加如下代码，通过USE_MYMATH控制使用哪个平方根函数：

```cpp
#ifdef USE_MYMATH
  const double outputValue = mysqrt(inputValue);
#else
  const double outputValue = sqrt(inputValue);
#endif
```
由于源代码现在需要USE_MYMATH，所以我们可以在TutorialConfig.h.in中添加如下代码 ：

```cpp
#cmakedefine USE_MYMATH
```
练习：为什么我们TutorialConfig.h.in文件中的USE_MYMATH在最后配置很重要？如果将两者倒置会发生什么？
运行cmake 可执行文件或 cmake-gui配置项目，然后使用您选择的构建工具构建它。然后运行构建的教程可执行文件。
还是和第一步教程一样进行构建：
![在这里插入图片描述](https://img-blog.csdnimg.cn/e8632c1478e146daa98a0fbb8367518b.png)

现在让我们更新USE_MYMATH的值。你在终端最简单的方法是使用 cmake-gui 或者 ccmake。或者，如果您想从命令行更改选项，请尝试：

```cpp
cmake ../Step2 -DUSE_MYMATH=OFF
```
重新构建并再次运行。
![在这里插入图片描述](https://img-blog.csdnimg.cn/d811c86f080e44c8859e3298d4aa0470.png)
