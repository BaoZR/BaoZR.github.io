---
layout: post
title:  windows下MSVC编译gtest并搭建cmake项目环境
date: 2022-04-07
categories:
- c/c++
tags: [c++]
---
- 编译器：VS2019
- 环境：win10
- googletest：1.11.0

先让我们把Googletest的库编出来

1.打开https://github.com/google/googletest要下载Release版本的源码
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-1.png)

2.根据官方提示修改一下googletest中的cmakelist
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-2.png)
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-3.png)

3.打开增强VS命令行
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-4.png)

4.cd到文件夹位置，根据官方提示输入。
（注：不用cd到那个googletest文件夹中，在最外层文件夹就行了）
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-5.png)
2022.4.7 补充
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-6.png)
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-7.png)
这样就搭好了生成库文件的sln了。

5.开始生成googletest的库
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-8.png)
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-9.png)

6.接下来要搭建cmake环境了，我这里就只考虑了debug版本，复制gtest_maind.lib,gtestd.lib,还有相关的头文件到项目目录下。

![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-10.png)

我这里简单点文件都放在一个地方。

![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-11.png)

7.编写一下Cmakelist和main函数，这种方法比较老了，还勉强能用。

![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-12.png)

```cmake
# project specific logic here.
#
cmake_minimum_required (VERSION 3.8)
set(CMAKE_CXX_STANDARD 11)

project ("310_algorithms_test")

Message(STATUS "CMAKE_BINARY_DIR:${CMAKE_BINARY_DIR}")
Message(STATUS "CMAKE_SOURCE_DIR:${CMAKE_SOURCE_DIR}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}) #设置程序输出目录为根目录a
link_directories(${CMAKE_SOURCE_DIR})
include_directories(${CMAKE_SOURCE_DIR}/include/gtest)
# Add source to this project's executable.
add_executable (${PROJECT_NAME} "310_algorithms_test.cpp" "310_algorithms_test.h")

# TODO: Add tests and install targets if needed.

```
**主函数**
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-13.png)

```c++
// CMakeProject1.cpp : Defines the entry point for the application.
//

#include "CMakeProject1.h"
#include "gtest/gtest.h"

#pragma comment(lib, "gtest_maind")
#pragma comment(lib, "gtestd")

using namespace std;

TEST(SomeTest, DoesThis) {
	EXPECT_EQ(4, 3);
}

int main()
{
	testing::InitGoogleTest();//此处为初始化
	RUN_ALL_TESTS();//执行所有测试用例
	cout << "Hello CMake." << endl;
	return 0;
}
```
运行结果是。。。
![](/images/post/compile-gtest-and-build-cmake-project/compile-gtest-and-build-cmake-project-14.png)


















