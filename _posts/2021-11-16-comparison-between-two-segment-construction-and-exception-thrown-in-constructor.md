---
layout: post
title:  二段构造与构造函数中抛出异常比较
date: 2021-11-16
categories:
- c/c++
tags: [c++]
---

&emsp;&emsp;有这么个情景，程序需要打开设备，而打开设备会失败。为了模拟这个情景，c++代码中写了个设备类，由于在进行其他操作之前需要先打开这个设备，所以这里要对类做点设计。经过查阅和分析，想了两种方法只调用一次就能够获得已经打开的设备对象。

&emsp;&emsp;一是二段构造，二是构造函数中抛异常。

#### 1. 二段构造
&emsp;&emsp;原来的构造函数中只是对内部变量进行赋值，construct函数负责完成打开设备操作，另外还要有个init静态函数把构造函数和construct函数包起来。在操作中发现，二段构造中的new操作似乎是不可避免的，这就需要在主函数中加入delete操作。
```c++
#include <iostream>
#include <Windows.h>

//模拟用来打开设备的函数，可能会失败,代表一个可能会失败的操作
int open(HANDLE* handle)
{
	srand(time(NULL));
	int num = rand() % 100;
	if (num < 50)
		return 0;
	else
		return -1;
}

class Device
{

public:

	Device() {
		handle = NULL;
	}
	~Device() {


	};
	bool construct() {//二段构造,这里只打开设备
		int ret = open(&handle);
		if (ret != 0)
		{
			std::cout << "open device fail" << " ret: " << ret << std::endl;
			return false;
		}
		std::cout << "open device success" << std::endl;
		return true;
	};

	static Device* init()//二段构造
	{
		Device* device = new Device();//这里的new看上去无法避免
		if (device->construct() == false)
		{
			delete device;
			return nullptr;
		}
		return device;
	}

private:
	HANDLE handle;//打开设备中往往有个句柄
};

int main(int argc, char* argv[])
{
	//Device* dev = Device::init();
	//delete dev;//之前的办法要多加一个关闭的操作
	
	//用一下这个方法就可以不用多加一步了
	auto ptr = std::unique_ptr<Device>{ Device::init() };

	//这里会有各种操作
	//....
}
```

#### 2. 构造加异常

&emsp;&emsp;在构造函数中加入try catch的处理异常的方法。由于构造函数没有返回值，拿到的对象不一定是成功打开了设备，还需要对内部变量handle进行判断。

```c++
#include <iostream>
#include <Windows.h>

//模拟用来打开设备的函数，可能会失败,代表一个可能会失败的操作
int open(HANDLE* handle)
{
	srand(time(NULL));
	int num = rand() % 100;
	if (num < 50)
		return 0;
	else
		return -1;
}

class Device
{

public:

	Device() {
		handle = NULL;
		try
		{//这里加异常判断
			int ret = open(&handle);
			if(ret !=0)
				throw std::string("can't open device\n");
		}
		catch (const std::string& str)
		{
			std::cout << str << std::endl;
		}
	}
	~Device() {


	};
	private:
	HANDLE handle;//打开设备中往往有个句柄
};

int main(int argc, char* argv[])
{
	Device dev;

	//这里是各种操作

	system("pause");
}

```
~~笔者还是更认可构造函数中加异常的操作，如果用二段操作，需要记得delete掉new出来的对象，这样就破坏了RAII。~~

两种方法都是可以的，前一种方法经过unique_ptr改进后，也是符合RALL的。





