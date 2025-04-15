---
layout: post
title:  vcpkg+cmake+vs使用静态库提示MSVCRTD与其他库使用冲突，请使用/NODEFAULTLIB:library
date: 2023-10-27
categories:
- c/c++
tags: [vcpkg,cmake]
---
![](/images/post/MSVCRTD-conflicts-1.png)
msvcrtd是MDd的，libcmtd是MTd的
要装的库fltk的x64-windows-static版本 都已经装了，为什么还会有冲突呢
![](/images/post/MSVCRTD-conflicts-2.png)
cmake中也设定了
![](/images/post/MSVCRTD-conflicts-3.png)
后来检查到导入的库，导入的是x64-windows的，为什么没有导入x64-windows-static呢
![](/images/post/MSVCRTD-conflicts-4.png)
最后在cmake-gui发现
![](/images/post/MSVCRTD-conflicts-5.png)
知道原因之后在cmake中加入
set(VCPKG_TARGET_TRIPLET x64-windows-static)，重新生成工程就行了