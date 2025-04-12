---
layout: post
title:  Graphics View中如何画一个可变大小位置的矩形(pyside)
date: 2025-02-20
categories:
- python
tags: [pyside,Graphics view]
---
**效果**

![](/images/post/deformable-rectangle.gif)

**说明**
1. 悬浮效果通过重写hoverMoveEvent实现，若鼠标在边框附近则修改鼠标样式，离开边框区域则恢复。
2. 移动和缩放怎么区分，鼠标按下去的时候，如果鼠标在边框附近，则是缩放，如果不在边框附近，则是移动。
3. 方法中有个代表图元位置和大小的Rect，操作通过adjust()/translate()修改到Rect,通过setRect()更新到图元中。

**代码**
```python
import typing

from PySide6.QtCore import Qt, QRectF, QSize
from PySide6.QtGui import QPen, QBrush, QColor, QResizeEvent
from PySide6.QtWidgets import (QApplication, QGraphicsView, QGraphicsScene,
                               QGraphicsItem, QGraphicsRectItem, QMainWindow,
                               QVBoxLayout, QWidget)


class ResizableRect(QGraphicsRectItem):
    def __init__(self,
                 rectSize = QRectF(0,0,100,200),
                 rectPen = QPen(QBrush(QColor('blue')), 5)
                 ):
        super().__init__()
        self.setFlag(QGraphicsItem.GraphicsItemFlag.ItemIsMovable, True)
        self.setPen(rectPen)
        self.setRect(rectSize)
        self.setAcceptHoverEvents(True)
        self.selected_edge = None
        #self.hover_edge = None
        self.click_pos = self.click_rect = None


    def mousePressEvent(self, event):
        """ The mouse is pressed, start tracking movement. """
        print('mousePress')
        self.click_pos = event.pos()
        rect = self.rect()
        if abs(rect.top() - self.click_pos.y()) < 5 and abs(rect.left() - self.click_pos.x()) < 5:
            self.selected_edge = 'topleft'
        elif abs(rect.top() - self.click_pos.y()) < 5 and abs(rect.right() - self.click_pos.x()) < 5:
            self.selected_edge = 'topright'
        elif abs(rect.bottom() - self.click_pos.y()) < 5 and abs(rect.left() - self.click_pos.x()) < 5:
            self.selected_edge = 'bottomleft'
        elif abs(rect.bottom() - self.click_pos.y()) < 5 and abs(rect.right() - self.click_pos.x()) < 5:
            self.selected_edge = 'bottomright'
        elif abs(rect.left() - self.click_pos.x()) < 5:
            self.selected_edge = 'left'
        elif abs(rect.right() - self.click_pos.x()) < 5:
            self.selected_edge = 'right'
        elif abs(rect.top() - self.click_pos.y()) < 5:
            self.selected_edge = 'top'
        elif abs(rect.bottom() - self.click_pos.y()) < 5:
            self.selected_edge = 'bottom'

        else:
            self.selected_edge = None
        self.click_pos = event.pos()
        self.click_rect = rect
        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
    	#如果已经设置标志不可移动，则直接返回
		flags = self.flags()
        if ((flags & QGraphicsItem.GraphicsItemFlag.ItemIsMovable) != QGraphicsItem.GraphicsItemFlag.ItemIsMovable):
             return super().mouseMoveEvent(event)
             
        """ Continue tracking movement while the mouse is pressed. """
        # Calculate how much the mouse has moved since the click.
        pos = event.pos()
        x_diff = pos.x() - self.click_pos.x()
        y_diff = pos.y() - self.click_pos.y()

        # Start with the rectangle as it was when clicked.
        rect = QRectF(self.click_rect)

        # Then adjust by the distance the mouse moved.
        if self.selected_edge is None:
            rect.translate(x_diff, y_diff)
        elif self.selected_edge == 'topleft':
            rect.adjust(x_diff, y_diff, 0, 0)
        elif self.selected_edge == 'topright':
            rect.adjust(0, y_diff, x_diff, 0)
        elif self.selected_edge == 'bottomleft':
            rect.adjust(x_diff, 0, 0, y_diff)
        elif self.selected_edge == 'bottomright':
            rect.adjust(0, 0, x_diff, y_diff)
        elif self.selected_edge == 'top':
            rect.adjust(0, y_diff, 0, 0)
        elif self.selected_edge == 'left':
            rect.adjust(x_diff, 0, 0, 0)
        elif self.selected_edge == 'bottom':
            rect.adjust(0, 0, 0, y_diff)
        elif self.selected_edge == 'right':
            rect.adjust(0, 0, x_diff, 0)

        print(self.selected_edge)

        # Also check if the rectangle has been dragged inside out.

        
        if rect.width() < 5:
            if self.selected_edge == 'left' or self.selected_edge == 'topleft' or self.selected_edge == "bottomleft":
                rect.setLeft(rect.right() - 5)
            elif self.selected_edge == 'right' or self.selected_edge == "topright" or self.selected_edge == "bottomright":
                rect.setRight(rect.left() + 5)
            
        if rect.height() < 5:
            if self.selected_edge == 'top' or self.selected_edge == "topleft" or self.selected_edge == "topright":
                rect.setTop(rect.bottom() - 5)
            elif self.selected_edge == 'bottom' or self.selected_edge == "bottomleft" or self.selected_edge == "bottomright":
                rect.setBottom(rect.top() + 5)


        # Finally, update the rect that is now guaranteed to stay in bounds.
        self.setRect(rect)

		#return super().mouseMoveEvent(event) 这句话不能加，不然失去变形效果
		
    def hoverMoveEvent(self, event):
        print('hoverMove')
        self.hover_pos = event.pos()
        rect = self.rect()
        cursor_shape = Qt.CursorShape.ArrowCursor
        
        #corner is not need to be hovered
        if abs(rect.left() - self.hover_pos.x()) < 5 and abs(rect.top() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeFDiagCursor
        elif abs(rect.right() - self.hover_pos.x()) < 5 and abs(rect.top() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeBDiagCursor
        elif abs(rect.left() - self.hover_pos.x()) < 5 and abs(rect.bottom() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeBDiagCursor
        elif abs(rect.right() - self.hover_pos.x()) < 5 and abs(rect.bottom() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeFDiagCursor
        elif abs(rect.left() - self.hover_pos.x()) < 5:
            cursor_shape = Qt.CursorShape.SizeHorCursor
        elif abs(rect.right() - self.hover_pos.x()) < 5:
            cursor_shape = Qt.CursorShape.SizeHorCursor
        elif abs(rect.top() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeVerCursor
        elif abs(rect.bottom() - self.hover_pos.y()) < 5:
            cursor_shape = Qt.CursorShape.SizeVerCursor        
        else:
            #self.selected_edge = None
            cursor_shape = Qt.CursorShape.ArrowCursor
        self.setCursor(cursor_shape)

        return super().hoverMoveEvent(event)

    def hoverLeaveEvent(self, event):
        print('hoverLeave')
        #self.selected_edge = None
        self.setCursor(Qt.CursorShape.ArrowCursor)
        return super().hoverLeaveEvent(event)
    

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        central = QWidget(self)
        self.setCentralWidget(central)

        self.rect = ResizableRect(rectSize=QRectF(30, 40,100, 70), rectPen=QPen(QBrush(QColor('red')), 5))
        scene = QGraphicsScene(0, 0, 300, 300)
        scene.addItem(self.rect)
        self.view = QGraphicsView(central)
        self.view.setScene(scene)

        layout = QVBoxLayout(central)
        self.setLayout(layout)
        layout.addWidget(self.view)

        #self.rect.setRect(QRectF(30, 40,100, 70))

def main():
    app = QApplication()
    window = MainWindow()
    window.show()

    app.exec_()


main()
```




