---
layout: post
title:  Graphics View中如何画一个可变形的椭圆(pyside)
date: 2025-02-20
categories:
- python
tags: [pyside,Graphics view]
---
**效果**：
![](/images/post/deformable-ellipse.gif)

**说明**

和之前写的可变形矩形相比，最大的差异是不用限制范围了，椭圆的width和height可以是负数。

**代码**
```python
#写一个可以移动的椭圆类，可以变型

from PySide6.QtCore import Qt, QRectF,  QPointF
from PySide6.QtGui import QPen, QBrush, QColor
from PySide6.QtWidgets import (QApplication, QGraphicsView, QGraphicsScene,
                               QGraphicsItem, QGraphicsEllipseItem, QMainWindow,
                               QVBoxLayout, QWidget)

class ResizableEllipse(QGraphicsEllipseItem):
    def __init__(self,
                 rect = QRectF(0, 0, 100, 50),
                 pen = QPen(QBrush(QColor('blue')), 5)
                 ):
        super().__init__()
        self.setRect(rect)
        self.setPen(pen)
        self.setFlag(QGraphicsItem.GraphicsItemFlag.ItemIsMovable)
        self.setAcceptHoverEvents(True)

        self.selected_point = None
        self.click_pos = None
        self.click_ellipse = None

    def mousePressEvent(self, event):
        self.click_pos = event.pos()      
        rect = self.rect()
        x , y , w, h = rect.x(), rect.y(), rect.width(), rect.height()
        top = QPointF(x + w/2, y)
        right = QPointF(x + w, y + h/2)
        bottom = QPointF(x + w/2, y + h)
        left = QPointF(x, y + h/2)

        if abs(self.click_pos.x() - top.x()) < 5 and abs(self.click_pos.y() - top.y()) < 5:
            self.selected_point = 'top'
           # print("top")
        elif abs(self.click_pos.x() - right.x()) < 5 and abs(self.click_pos.y() - right.y()) < 5:
            self.selected_point = 'right'
        elif abs(self.click_pos.x() - bottom.x()) < 5 and abs(self.click_pos.y() - bottom.y()) < 5:
            self.selected_point = 'bottom'
        elif abs(self.click_pos.x() - left.x()) < 5 and abs(self.click_pos.y() - left.y()) < 5:
            self.selected_point = 'left'
        else:
            self.selected_point = None

        self.click_ellipse = self.rect()
        return super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
		#如果已经设置标志不可移动，则直接返回
		flags = self.flags()
        if ((flags & QGraphicsItem.GraphicsItemFlag.ItemIsMovable) != QGraphicsItem.GraphicsItemFlag.ItemIsMovable):
             return super().mouseMoveEvent(event)
		
        pos = event.pos()
        x_diff = pos.x() - self.click_pos.x()
        y_diff = pos.y() - self.click_pos.y()
        print("x_diff:" + str(x_diff) , "y_diff:" + str(y_diff))    
        rect = QRectF(self.click_ellipse)

        if self.selected_point == None:
            rect.translate(x_diff, y_diff)
        elif self.selected_point == 'top':
            rect.adjust(0,y_diff,0,0)
        elif self.selected_point == 'right':
            rect.adjust(0,0,x_diff,0)
        elif self.selected_point == 'bottom':
            rect.adjust(0,0,0,y_diff)
        elif self.selected_point == 'left':
            rect.adjust(x_diff,0,0,0)

        self.setRect(rect)
        print(rect.x(), rect.y(), rect.width(), rect.height())
        # return super().mouseMoveEvent(event) 这句话不能加


    def hoverMoveEvent(self, event):
        pos = event.pos()
        rect = self.rect()
        x , y , w, h = rect.x(), rect.y(), rect.width(), rect.height()
        top = QPointF(x + w/2, y)
        right = QPointF(x + w, y + h/2)
        bottom = QPointF(x + w/2, y + h)
        left = QPointF(x, y + h/2)
        cursor_shape = Qt.CursorShape.ArrowCursor

        if abs(pos.x() - top.x()) < 5 and abs(pos.y() - top.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeVerCursor
        elif abs(pos.x() - right.x()) < 5 and abs(pos.y() - right.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeHorCursor
        elif abs(pos.x() - bottom.x()) < 5 and abs(pos.y() - bottom.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeVerCursor
        elif abs(pos.x() - left.x()) < 5 and abs(pos.y() - left.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeHorCursor
        else:
            cursor_shape = Qt.CursorShape.ArrowCursor

        self.setCursor(cursor_shape)
        return super().hoverMoveEvent(event)
    
    def hoverLeaveEvent(self, event):
        self.setCursor(Qt.CursorShape.ArrowCursor)
        return super().hoverLeaveEvent(event)

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        central = QWidget(self)
        self.setCentralWidget(central)

        self.ellipse = ResizableEllipse()
        scene = QGraphicsScene(0, 0, 300, 300)
        scene.addItem(self.ellipse)
        self.view = QGraphicsView(central)
        self.view.setScene(scene)

        layout = QVBoxLayout(central)
        self.setLayout(layout)
        layout.addWidget(self.view)


def main():
    app = QApplication()
    window = MainWindow()
    window.show()

    app.exec_()

main()
```
