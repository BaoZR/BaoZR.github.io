---
layout: post
title:  QTableWidget如何实现点击表格空白处取消选中行（pyside）
date: 2024-11-14
categories:
- python
tags: [pyside]
---
考虑过使用QTableWidget的selectedItems()方法，如果点击的item内容为空，则方法返回的结果是空的，这样不能和点击table空白处区分。
最后用安装事件过滤器的方法实现了，用到了viewport,QModelIndex，看代码中的注释

**实现效果**
![](/images/post/uncheck-the-row.gif)

**代码**
```python
# 主函数处创建事件过滤器实例，要安装在viewport上，viewport可以响应到点击事件
# TableWidget响应不到点击事件
self.event_filter = TableWidgetFilter(self.ui.headersTable)
self.ui.headersTable.viewport().installEventFilter(self.event_filter)

....

class TableWidgetFilter(QObject):
    def __init__(self, parent=None):
        super(TableWidgetFilter, self).__init__(parent)
    
    def eventFilter(self, watched, event):
        if event.type() == QEvent.MouseButtonRelease:
            # parentWidget()方法获取TableWidget,indexAt获取的是QModelIndex
            # QModelIndex 可以获取当前点击的行
            idx = watched.parentWidget().indexAt(event.position().toPoint())
            if(idx.row() < 0):
                watched.parentWidget().setCurrentItem(None)
        return super(TableWidgetFilter, self).eventFilter( watched, event)

```

