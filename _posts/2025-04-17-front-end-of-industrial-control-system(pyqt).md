---
layout: post
title:  工控系统前端设计
date: 2025-04-17
categories:
- python
tags: [pyqt,Graphics view]
---
题目源自：白月黑羽的项目实战四-[[工控系统前端](https://www.byhy.net/py/qt/proj-prac4/)]
![](/images//post/industrial-front-end/industrial-front-end-1.png)

代码已上传至gitcode [https://gitcode.com/m0_37662818/Industrial_Control_System_Front_End](https://gitcode.com/m0_37662818/Industrial_Control_System_Front_End)

心得体会：直接用组态软件或者js吧

**项目亮点**
1. tablemodel的使用，绑定了表格和数据
2. 风机自定义item的实现

对比原题的要求，列了下表格，实现的部分都打钩。

| 功能点 |	是否完成✔❌|备注|
| :-----:     |  :---: |  :----: |
| 布局	  |   ✔|	网站已给解答|
| 图标拖放|	✔	|网站已给解答|
| 各类 item 支持键盘方向键移动|	✔	|多选删除多个 item 未实现|
| 点击 item，表格显示|	✔	|
| 表格修改，改变 item	|✔|	
| 图标工作栏	|✔	| 打开/保存/删除/清空
| 后端服务器	|✔|	自研，网站未提供|
| 连接服务器 断线重连|	✔	|
| item 增加设备编号	|✔|	
| 摄像头	|❌|	该功能未实现
| 显示服务器通知消息|	✔|	
| 水缸	|❌|	该 item 未实现|
| 只读模式	|✔|	
| 实时统计图	|❌|	
| 风泵	|✔|	消息确认和重发未实现|

**实现效果**

可以看到修改风机转速后，风速检测值的变化。各种自定义item也都有属性可以交互。
![](/images/post/industrial-front-end/industrial-front-end-2.gif)

服务端上设备编号是固定的，根据题目是 

|设备|编号|
|:---:|:---:|
|毒气监测仪| aaaa0001|
||aaaa0021|
|温度湿度计|aaaa0002|
||aaaa0022|
|空气流量仪|aaaa0003|
||aaaa0023|
|水流量表|aaaa0004|
||aaaa0024|

（饮用水箱、摄像头除外）
