---
layout: post
title:  基于C语言封装的工具类库-关于一些文件操作和图片操作
date: 2021-12-09
categories:
- c/c++
tags: [c,tool]
---
#### **utils**
这是一个工作中用到的，基于C语言封装的工具类库，用于MSVC编译器，可以在WINDOWS环境下使用。

#### **函数介绍：**

|函数名     | 介绍|
|-------- | ----- |
|write_data_hex| 向文件写指定大小的数据|
|read_data_hex|  从文件读指定大小的数据|
|list_matched_filename_app  | 枚举相对app路径下匹配到的文件名|
| print_matched_filename_app | 打印相对app路径下匹配到的文件名|
|auto_list_filename_app  |  一个相对自动化的函数，可以枚举相对app路径下匹配到的文件名,要记得释放name_list中分配的内存|
| save_gray_bmp_app | 创建相对app路径文件的图片|
|count_filename_app | 统计相对app路径文件下文件的数量|
|create_dir_in_app | 在APP中建立文件夹|
|mkdirs | 生成指定路径的文件夹|
|get_app_path | 获取app所在文件夹路径|
|add8GreyBmpHead | 给未加工的原始数据加上一个8位灰度图的头|
|add8GreyBmpHead2File | 将原始数据加上8位灰度头，并输出到一个文件中|

github地址：[https://github.com/BaoZR/c_utils](https://github.com/BaoZR/c_utils/)
