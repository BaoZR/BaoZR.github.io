---
layout: post
title:  C语言宏定义格式化控制台打印
date: 2024-07-18
categories:
- C
tags: [C]
---

写了个简单的控制台打印代码，有三种打印级别 DEBUG INFO ERROR，支持颜色打印，支持时间打印
在MSVC环境中使用

**实现效果**
![](/images/post/format-console-debug.png)

代码如下
```c

#include <time.h>
#include <string.h>
#include <stdio.h>

// log level 
#define LOG_LEVEL_DEBUG  (1)
#define LOG_LEVEL_INFO   (2)
#define LOG_LEVEL_ERROR  (3)

// log config 
#define LOG_OUTPUT_LEVEL LOG_LEVEL_DEBUG

// log system component
#define LOG_COLOR_RED       "\033[31;1m"
#define LOG_COLOR_GREEN     "\033[32;1m"
#define LOG_COLOR_YELLOW    "\033[33m"
#define LOG_COLOR_BLUE      "\033[34;1m"
#define LOG_COLOR_CARMINE   "\033[35m"
#define LOG_COLOR_CYAN      "\033[36;1m"
#define LOG_COLOR_WHITE     "\033[37m"
#define LOG_COLOR_DEFAULT
#define LOG_COLOR_END       "\033[m"

#define LOG_BASE_FILENAME \
    (strrchr(__FILE__, '/') ? strrchr(__FILE__, '/') + 1 : \
    strrchr(__FILE__, '\\') ? strrchr(__FILE__, '\\') + 1 : __FILE__)


#define LOG(prefix,format,...)												    \
		do{                                                                     \
				time_t t;														\
				struct tm ti;													\
				time(&t);														\
				localtime_s(&ti,&t);											\
				printf(""prefix" [%d:%02d:%02d] (%s:%d %s) "format"\n",         \
						ti.tm_hour,ti.tm_min,ti.tm_sec,						    \
						LOG_BASE_FILENAME,__LINE__,__FUNCTION__,##__VA_ARGS__); \
		} while (0)

#if LOG_LEVEL_DEBUG >= LOG_OUTPUT_LEVEL
#define LOG_DEBUG(fmt,...) LOG(LOG_COLOR_GREEN "[DEBUG]", fmt LOG_COLOR_END,##__VA_ARGS__)
#else
#define LOG_DEBUG(fmt,...)	((void)0)
#endif

#if LOG_LEVEL_INFO >= LOG_OUTPUT_LEVEL
#define LOG_INFO(fmt,...) LOG(LOG_COLOR_WHITE "[INFO]", fmt LOG_COLOR_END,##__VA_ARGS__)
#else
#define LOG_INFO(fmt,...)	((void)0)
#endif

#if  LOG_LEVEL_ERROR >= LOG_OUTPUT_LEVEL
#define LOG_ERROR(fmt,...) LOG(LOG_COLOR_RED "[ERROR]",fmt LOG_COLOR_END,##__VA_ARGS__)
#else
#define LOG_DEBUG(fmt,...)	((void)0)
#endif


int main(){
    LOG_DEBUG("Hello World!");
    LOG_INFO("Hello World!");
    LOG_ERROR("Hello World!");
    return 0;
}
```




