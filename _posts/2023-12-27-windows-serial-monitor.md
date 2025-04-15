---
layout: post
title:  windows系统下监控串口设备的插入和拔出（C++）
date: 2023-12-27
categories:
- c/c++
tags: [windows,serial device]
---

项目github地址：https://github.com/BaoZR/serial_monitor

### **思路**
特别感谢 [Hardware Change Detection](https://www.codeproject.com/Articles/119168/Hardware-Change-Detection)
要想获得设备的插入和拔出情况，就要借助windows自带的消息循环，通过注册串口类变动的通知，来实现这一目标。
设计思路如下图
![](/images/post/serial_monitor-1.png)
最后我写了一个演示效果的ui-demo和一个方便使用的lib导出库。
### **演示效果**
![](/images/post/serial_monitor-2.gif)
### **代码(c++)**

**lib导出库的头文件**
```c++
/**
 * @file serial_monitor_lib.h
 * @brief 用于监控串口物理插入和物理拔出的库，只能在win32下使用
 * @details 监控窗口为固定名字的窗口，所以请勿重复调用。
 * @author cuicui
 * @date 2023-09-04
 * @version 0.1
 * @copyright MIT
 * @web https://github.com/BaoZR/serial_monitor
 */



#ifndef _SERIAL_MONITOR_LIB_H_
#define _SERIAL_MONITOR_LIB_H_

#include <string>

#ifdef SERIAL_MONITOR_LIB_EXPORTS
#ifdef _WIN32
#define SERIAL_MONITOR_API __declspec(dllexport) __stdcall
#else
#define SERIAL_MONITOR_API __attribute__((visibility("default")))
#endif // _WIN32
#else
#ifdef _WIN32
#define SERIAL_MONITOR_API __declspec(dllimport) __stdcall
#else
#define SERIAL_MONITOR_API
#endif // _WIN32
#endif // SERIAL_MONITOR_LIB_EXPORTS

#define IN
#define OUT
#define INOUT


#define DEVICE_NEW                              (1)     /* 新的设备 */
#define DEVICE_DELETE                           (2)     /* 被删除的设备 */

typedef std::string DeviceId;

#ifdef __cplusplus
extern "C" {
#endif // __cplusplus

/**
 * @brief 监控的状态回调函数，用于初始化，串口设备插入，串口设备拔出时，返回状态，需传入monitor_init函数中
 * @param id 串口设备的ID
 * @param status 发现新的设备或者该设备已删除
 *               DEVICE_NEW 
 *               DEVICE_DELETE
 * @param friendly_name 发现新的设备时返回友好的串口名字，删除设备时返回之前友好的串口名字
 */
typedef void (*device_change_progress)(
    IN const std::string& id,
    IN int status,
    IN const std::string& friendly_name);

/**
 * @brief 用于初始化监听线程，监听是单窗口，请勿重复调用
 * @param device_change_progress 传入的回调函数
 *  
 */
void SERIAL_MONITOR_API monitor_init(
    IN device_change_progress progress_cb
);

/** @brief 用于关闭监听
 * 
 */
void SERIAL_MONITOR_API monitor_terminate();


#ifdef __cplusplus
}
#endif // __cplusplus

#endif

```
**lib导出库的源文件**

```c++
#define SERIAL_MONITOR_LIB_EXPORTS
#include "serial_monitor_lib.h"
#include "simple_notify_window.h"
#include "enum_device.h"
#include "device_info.h"
#include <iostream>
#include <mutex>

class DeviceChange : public IDeviceChanged
{
public:
    explicit DeviceChange(device_change_progress progress_cb);
    void InterfaceArrival(const GUID &guid);
    void InterfaceRemoved(const std::string &lower_dbcc);

private:
    device_change_progress progress_cb_;
};

static std::unique_ptr<SimpleNotifyWindow> window_ = nullptr;
static std::unique_ptr<DeviceChange> device_change_ = nullptr;
static HDEVNOTIFY h_dev_notify_ = nullptr;
static std::map<DeviceId, DeviceInfo> actual_devices_;
static bool init_flag_ = false;
static std::mutex section_;

DeviceChange::DeviceChange(device_change_progress progress_cb)
{
    this->progress_cb_ = progress_cb;
};
void DeviceChange::InterfaceRemoved(const std::string &lower_dbcc)
{
    section_.lock();
    if(actual_devices_.count(lower_dbcc) != 0)
    {
        std::string name = actual_devices_[lower_dbcc].GetFriendlyName();
        actual_devices_.erase(lower_dbcc);
        this->progress_cb_(lower_dbcc, DEVICE_DELETE, name);
    }
    section_.unlock();
}
void DeviceChange::InterfaceArrival(const GUID &guid)
{
    std::vector<DeviceInfo> temp_devices = detect_device::GetDevicesByGuid(&const_cast<GUID&>(guid));
    for(std::vector<DeviceInfo>::const_iterator iter = temp_devices.begin();iter != temp_devices.end();iter++)
    {
        section_.lock();
        if(actual_devices_.count((*iter).GetDbcc()) == 0)
        {
            actual_devices_[(*iter).GetDbcc()] = *iter;
            this->progress_cb_((*iter).GetDbcc(), DEVICE_NEW, (*iter).GetFriendlyName());
        }
        section_.unlock();
    }
}

void SERIAL_MONITOR_API monitor_init(
    IN device_change_progress progress_cb)
{
    if (init_flag_ == true)
    {
        monitor_terminate();
    }
    device_change_.reset(new DeviceChange(progress_cb));
    window_.reset(new SimpleNotifyWindow(*device_change_));
    h_dev_notify_ = detect_device::RegisterNotify(GUID_DEVINTERFACE_COMPORT, (*window_).GetHandle());
    init_flag_ = true;
    (*device_change_).InterfaceArrival(GUID_DEVINTERFACE_COMPORT);//初始化扫描

};

void SERIAL_MONITOR_API monitor_terminate()
{
    if (init_flag_ == true)
    {
        detect_device::UnRegisterNotify(h_dev_notify_);
        window_.reset(nullptr);
        device_change_ = nullptr;
        actual_devices_.clear();
        init_flag_ = false;
    }
};

```


