---
layout: post
title:  springboot事务异常Transaction rolled back because it has been marked as rollback-only
date: 2023-12-15
categories:
- java
tags: [spring]
---
![](/images/post/transaction-exception-1.png)
使用junit单元测试异常用例的时侯，打印报事务异常，通过查看源码，提示是一个全局的rollback标记导致的。也查阅了些相关资料，知道事务回滚有传播性，默认的回滚事务会从里面传播到外面。然后在对应的方法加上(propagation=Propagation.NOT_SUPPORTED)，限制事务传播范围，再运行测试就没有异常报错了。
![](/images/post/transaction-exception-2.png)
