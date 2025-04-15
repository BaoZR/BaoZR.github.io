---
layout: post
title:  vs2019 控制台出现乱码的解决记录
date: 2022-02-21
categories:
- c/c++
tags: [c]
---
#### 情况：
- 当时vs中的c语言程序以utf-8的格式存储，而windows控制台的默认输出为GBK，两者不统一，这样就导致控制台的中文显示为乱码。

#### 解决方法：
1. 使用SetConsoleOutputCP(65001)函数，这样控制台的编码输出就为UTF-8了，编码相同之后，输出的内容就没有乱码了。
2. 在命令行中（Command Line） 加入/source-charset:utf-8 /execution-charset:GBK，这样生成的控制台的编码格式就是GBK的。[MSDN对/execution-charset的解释](https://docs.microsoft.com/zh-cn/cpp/build/reference/execution-charset-set-execution-character-set?view=msvc-170
)
3. 使用自定义的UTF-8转GBK的函数，保证字符串输出的时候是GBK的。[别人写的转换函数](https://blog.csdn.net/weixin_43333380/article/details/108010546?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1.pc_relevant_default&spm=1001.2101.3001.4242.2&utm_relevant_index=3)

```c
//网上找的函数
std::string GBKToUTF8(const std::string& strGBK)
{
    std::string strOutUTF8 = "";
    WCHAR* str1;//用来放gbk
    int n = MultiByteToWideChar(CP_ACP, 0, strGBK.c_str(), -1, NULL, 0);
    str1 = new WCHAR[n];
    MultiByteToWideChar(CP_ACP, 0, strGBK.c_str(), -1, str1, n);//放宽字符的gbk，难道只有放到宽字符中才能转换吗 //好像是的
    n = WideCharToMultiByte(CP_UTF8, 0, str1, -1, NULL, 0, NULL, NULL);
    char* str2 = new char[n];//用来放utf8
    WideCharToMultiByte(CP_UTF8, 0, str1, -1, str2, n, NULL, NULL);//不知为何放在多字节的utf8中
    strOutUTF8 = str2;//char直接赋值给String
    delete[]str1;
    str1 = NULL;
    delete[]str2;
    str2 = NULL;
    return strOutUTF8;
}
//自己模仿的
std::string UTF8ToGBK(const std::string&strUTF8) {
    std::string strOutGBK = "";
    WCHAR* wstr;
    int n = MultiByteToWideChar(CP_UTF8, 0, strUTF8.c_str(), -1, NULL, 0);
    wstr = new WCHAR[n];
    MultiByteToWideChar(CP_UTF8, 0, strUTF8.c_str(), -1, wstr, n);

    n = WideCharToMultiByte(CP_ACP, 0, wstr, -1, NULL, 0, NULL, NULL);
    char* str = new char[n];
    WideCharToMultiByte(CP_ACP, 0, wstr, -1, str, n, NULL, NULL);
    strOutGBK = str;
    delete[] wstr;
    wstr = NULL;
    delete[] str;
    str = NULL;
    return strOutGBK;
}
```
