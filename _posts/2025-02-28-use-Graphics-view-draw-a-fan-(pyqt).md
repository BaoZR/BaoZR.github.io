---
layout: post
title:  Graphics View画一个可调速的风机(pyqt)
date: 2025-02-28
categories:
- python
tags: [pyqt,Graphics view]
---
效果如图：
![](/images/post/fan.gif)
&emsp;&emsp;风机具备调节转速的功能，转速通过扇叶旋转的快慢来区别，共分为四档，其中零档为静止状态，而一、二、三档则依次增加转速。在代码中，BlowerWrapper 类包含了可旋转的扇叶、风机外框以及选项三个主要部分。此处有两处关键点值得注意：

&emsp;&emsp;BlowerWrapper 选择继承 QObject 的主要原因是为了配合 QPropertyAnimation 的使用，由于普通的 QGraphicsItem 并未继承 QObject，无法使用 QPropertyAnimation 增加动画功能。

&emsp;&emsp;fan_item 、option （包括 option1、option2、option3）均使用 setParentItem 方法将 blower_base 设置为它们的父对象，这样便于它们随父对象一起移动。之所以没有使用 group ，是因为使用 group 后，item 将无法接收到鼠标的点击事件。见<https://stackoverflow.com/questions/60476803/how-to-propagate-mouse-events-to-a-qgraphicsitem-in-a-qgraphicsitemgroup>

&emsp;&emsp;下面是代码，同步于gitcode <https://gitcode.com/m0_37662818/fan>

```python
from PyQt5.QtCore import *
from PyQt5.QtCore import pyqtProperty
from PyQt5.QtGui import *
from PyQt5.QtWidgets import *

import sys


class MySignal(QObject):
    fan_singal = pyqtSignal(int)

class clickedRectItem(QGraphicsRectItem):
    def __init__(self, level = None, parent=None):
        super().__init__(parent)
        self.level = level
        self.signal = MySignal()

    def mousePressEvent(self, event):
        if event.button() == Qt.MouseButton.LeftButton:

            if self.level is not None:
                self.signal.fan_singal.emit(self.level)
        return super().mousePressEvent(event)

class BlowerWrapper(QObject):

    def __init__(self, parent=None):
        super().__init__(parent)
        fan_pixmap = QPixmap("./fan.png")
        blower_frame_pixmap = QPixmap("./blower_frame.png")
        self.fan_item = QGraphicsPixmapItem(fan_pixmap)
        self.blower_base = QGraphicsPixmapItem(blower_frame_pixmap)

        self.option1 = clickedRectItem(level=1)
        self.option2 = clickedRectItem(level=2)
        self.option3 = clickedRectItem(level=3)

        self.fan_item.setParentItem(self.blower_base)
        self.option1.setParentItem(self.blower_base)
        self.option2.setParentItem(self.blower_base)
        self.option3.setParentItem(self.blower_base)

        self.fan_item.setTransformOriginPoint(self.fan_item.boundingRect().center())
        self.cal_rect()
        self.animation_init()
        self.option1.signal.fan_singal.connect(self.animate)
        self.option2.signal.fan_singal.connect(self.animate)
        self.option3.signal.fan_singal.connect(self.animate)
        self.blower_base.setFlag(QGraphicsItem.GraphicsItemFlag.ItemIsMovable, True)
        self.blower_base.setFlag(QGraphicsItem.GraphicsItemFlag.ItemIsSelectable, True)
        self.blower_base.setShapeMode(QGraphicsPixmapItem.ShapeMode.BoundingRectShape)
        self.level = 0

    def animation_init(self):
        self.anim = QPropertyAnimation(self, b'rotation')
        self.anim.setDuration(1600)
        self.anim.setStartValue(0)
        self.anim.setEndValue(360)
        self.anim.setLoopCount(-1)

    def animate(self,level:int = None):
        if level is None or level == self.level or level == 0:
            self.anim.stop()
            self.set_rect_color(0)
            self.level = 0
            return
        else:
            level = int(level)
            self.level = level
            if level == 1:
                time = 1600
            elif level == 2:
                time = 700
            elif level == 3:
                time = 300
            else:
                return
            self.anim.setDuration(time)
            self.set_rect_color(self.level)
            self.anim.start()

    def cal_rect(self):
        x_len = self.blower_base.boundingRect().width()
        y_len = self.blower_base.boundingRect().height()
       
        rect_side_length = x_len / 4
        x_space = (x_len - rect_side_length * 3 ) / 2
        x1 = 0.0 
        y1 = y_len + x_space / 2
        x2 = rect_side_length + x_space
        y2 = y_len + x_space / 2
        x3 = rect_side_length * 2 + x_space * 2
        y3 = y_len + x_space / 2
        self.option1.setRect(QRectF(x1, y1, rect_side_length, rect_side_length))
        self.option2.setRect(QRectF(x2, y2, rect_side_length, rect_side_length))
        self.option3.setRect(QRectF(x3, y3, rect_side_length, rect_side_length))

    def _set_rotation(self, angle):
        self.fan_item.setRotation(angle)
  
    def set_rect_color(self, level:int):
        for item in [self.option1, self.option2, self.option3]:
                item.setBrush(QColor(255, 255, 255, 255))
                item.setPen(QColor(0,0,0,255))

        if level == 1:
            self.option1.setBrush(QColor(0, 128, 255, 255))
            self.option1.setPen(QColor(0, 128, 255, 255))
        elif level == 2:
            self.option2.setBrush(QColor(0, 128, 255, 255))
            self.option2.setPen(QColor(0, 128, 255, 255))
        elif level == 3:
            self.option3.setBrush(QColor(0, 128, 255, 255))
            self.option3.setPen(QColor(0, 128, 255, 255))

    rotation = pyqtProperty(int, fset=_set_rotation)

class MainWindow(QGraphicsView):
    def __init__(self):
        super().__init__()
        self.scene_area =  QGraphicsScene(self)
        self.scene_area.setSceneRect(0,0,300,300)
        self.setScene(self.scene_area)
        self.show()

        self.blower_wrapper = BlowerWrapper()
        self.scene_area.addItem(self.blower_wrapper.blower_base)

        self.blower_wrapper.blower_base.setPos(125,110)
        self.btn = QPushButton("开始")

if __name__ == '__main__':
    app = QApplication(sys.argv)
    ex = MainWindow()
    sys.exit(app.exec_())

```