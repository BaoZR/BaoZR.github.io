---
layout: post
title:  关于仿写类postman功能软件题目的解答
date: 2024-11-15
categories:
- python
tags: [pyside]
---

**原题：**
![](/images/post/imitative-postman-software-1.png)
**使用效果：**
![](/images/post/imitative-postman-software-2.gif)
**解答：**
```python
from PySide6.QtWidgets import QApplication, QMessageBox,QTableWidgetItem,QHeaderView,QWidget,QTableWidget
from PySide6.QtCore import QEvent,QObject
from PySide6.QtUiTools import QUiLoader
import time
import requests

uiLoader = QUiLoader()

class TableWidgetFilter(QObject):
    def __init__(self, parent=None):
        super(TableWidgetFilter, self).__init__(parent)
    
    def eventFilter(self, watched, event):
        if event.type() == QEvent.MouseButtonRelease:
            idx = watched.parentWidget().indexAt(event.position().toPoint())
            if(idx.row() < 0):
                watched.parentWidget().setCurrentItem(None)
        return super(TableWidgetFilter, self).eventFilter( watched, event)

# 用于打印请求消息， 参数为 PreparedRequest 对象
def pretty_print_request(req):
    if req.body == None:
        msgBody = ''
    else:
        msgBody = req.body
    
    # 打印请求消息，如果为空不拼接进去
    result = '\n----------- 发送请求 -----------'
    if req.method!= None and req.url!= None:
        result = result + '\n' + req.method + ' ' + req.url 
    if req.headers != None and len(req.headers) > 0:
        result = result +  '\n' 
        result = result + str('\n'.join('{}: {}'.format(k, v) for k, v in req.headers.items()))
    if msgBody!= '':
        result = result + str('\n' + msgBody)
    return result

# 用于打印响应消息
def pretty_print_response(res) -> str:
    result = '\n----------- 得到响应 -----------'
    if res.status_code!= None:
        result = result + str('\nHTTP/1.1 '+ str(res.status_code))
    if res.headers!= None and len(res.headers) > 0:
        result = result + '\n' 
        result = result + str('\n'.join('{}: {}'.format(k, v) for k, v in res.headers.items()))
    if res.text!= '':
        result = result + str('\n' + res.text)
    return result


class MainWindow:
    def __init__(self):
        self.ui = uiLoader.load('.\\main.ui')

        self.ui.addBtn.clicked.connect(self.add_header)
        self.ui.removeBtn.clicked.connect(self.remove_header)
        self.ui.headersTable.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        
        # 创建事件过滤器实例
        self.event_filter = TableWidgetFilter(self.ui.headersTable)
        self.ui.headersTable.viewport().installEventFilter(self.event_filter)
        self.ui.clearBtn.clicked.connect(self.clear_result)
        self.ui.sendBtn.clicked.connect(self.send)

    def add_header(self):
        self.ui.headersTable.insertRow(self.ui.headersTable.rowCount())
    
    def remove_header(self):
        row = self.ui.headersTable.currentRow()
        if row >= 0:
            self.ui.headersTable.removeRow(row)
            self.ui.headersTable.setCurrentItem(None)
    
    def send(self):
        #获取多个数据源
        request_type = self.ui.requestBox.currentText()
        request_url = self.ui.urlEdit.text()
        request_headers = {}
        for i in range(self.ui.headersTable.rowCount()):
            if self.ui.headersTable.item(i, 0) == None or self.ui.headersTable.item(i, 1) == None:
                continue
            key = self.ui.headersTable.item(i, 0).text()
            value = self.ui.headersTable.item(i, 1).text()
            request_headers[key] = value
        
        request_body = self.ui.bodyEdit.toPlainText()
        try:
            req = requests.Request(request_type, 
                            request_url, 
                            headers=request_headers, 
                            data=request_body)      
            prepared = req.prepare()
        except requests.exceptions.RequestException as e:
            QMessageBox.warning(self.ui, "错误", "请求参数错误\n" + str(e))
            return

        self.ui.resultText.appendPlainText(pretty_print_request(prepared))
        #发送包
        try:
            res = requests.Session().send(prepared)
            self.ui.resultText.appendPlainText(pretty_print_response(res))
        except requests.exceptions.RequestException as e:
            QMessageBox.warning(self.ui, "错误", "请求失败\n" + str(e))
            return

    def clear_result(self):
        self.ui.resultText.clear()


app = QApplication([])
mainWindow = MainWindow()
mainWindow.ui.show()
app.exec()

```
动态加载的main.ui文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>Form</class>
 <widget class="QWidget" name="Form">
  <property name="geometry">
   <rect>
    <x>0</x>
    <y>0</y>
    <width>534</width>
    <height>519</height>
   </rect>
  </property>
  <property name="windowTitle">
   <string>HTTP接口测试</string>
  </property>
  <layout class="QVBoxLayout" name="verticalLayout_3">
   <item>
    <layout class="QVBoxLayout" name="verticalLayout_2">
     <item>
      <layout class="QHBoxLayout" name="horizontalLayout">
       <item>
        <widget class="QComboBox" name="requestBox">
         <item>
          <property name="text">
           <string>GET</string>
          </property>
         </item>
         <item>
          <property name="text">
           <string>POST</string>
          </property>
         </item>
        </widget>
       </item>
       <item>
        <widget class="QLineEdit" name="urlEdit"/>
       </item>
       <item>
        <widget class="QPushButton" name="sendBtn">
         <property name="text">
          <string>发送</string>
         </property>
        </widget>
       </item>
      </layout>
     </item>
     <item>
      <widget class="Line" name="line">
       <property name="orientation">
        <enum>Qt::Horizontal</enum>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QSplitter" name="splitter_3">
       <property name="orientation">
        <enum>Qt::Horizontal</enum>
       </property>
       <widget class="QSplitter" name="splitter_2">
        <property name="orientation">
         <enum>Qt::Vertical</enum>
        </property>
        <widget class="QSplitter" name="splitter">
         <property name="orientation">
          <enum>Qt::Horizontal</enum>
         </property>
         <widget class="QLabel" name="label">
          <property name="font">
           <font>
            <pointsize>12</pointsize>
           </font>
          </property>
          <property name="layoutDirection">
           <enum>Qt::LeftToRight</enum>
          </property>
          <property name="text">
           <string>消息头</string>
          </property>
          <property name="alignment">
           <set>Qt::AlignCenter</set>
          </property>
         </widget>
         <widget class="QPushButton" name="addBtn">
          <property name="sizePolicy">
           <sizepolicy hsizetype="Fixed" vsizetype="Fixed">
            <horstretch>0</horstretch>
            <verstretch>0</verstretch>
           </sizepolicy>
          </property>
          <property name="maximumSize">
           <size>
            <width>50</width>
            <height>16777215</height>
           </size>
          </property>
          <property name="text">
           <string>+</string>
          </property>
         </widget>
         <widget class="QPushButton" name="removeBtn">
          <property name="sizePolicy">
           <sizepolicy hsizetype="Fixed" vsizetype="Fixed">
            <horstretch>0</horstretch>
            <verstretch>0</verstretch>
           </sizepolicy>
          </property>
          <property name="maximumSize">
           <size>
            <width>50</width>
            <height>16777215</height>
           </size>
          </property>
          <property name="text">
           <string>-</string>
          </property>
         </widget>
        </widget>
        <widget class="QTableWidget" name="headersTable">
         <column>
          <property name="text">
           <string>名称</string>
          </property>
         </column>
         <column>
          <property name="text">
           <string>值</string>
          </property>
         </column>
        </widget>
       </widget>
       <widget class="Line" name="line_2">
        <property name="minimumSize">
         <size>
          <width>15</width>
          <height>0</height>
         </size>
        </property>
        <property name="orientation">
         <enum>Qt::Vertical</enum>
        </property>
       </widget>
       <widget class="QWidget" name="layoutWidget_2">
        <layout class="QVBoxLayout" name="verticalLayout">
         <item>
          <widget class="QLabel" name="label_2">
           <property name="font">
            <font>
             <pointsize>12</pointsize>
            </font>
           </property>
           <property name="text">
            <string>消息体</string>
           </property>
           <property name="alignment">
            <set>Qt::AlignCenter</set>
           </property>
          </widget>
         </item>
         <item>
          <widget class="QPlainTextEdit" name="bodyEdit">
           <property name="enabled">
            <bool>true</bool>
           </property>
          </widget>
         </item>
        </layout>
       </widget>
      </widget>
     </item>
     <item>
      <widget class="Line" name="line_3">
       <property name="orientation">
        <enum>Qt::Horizontal</enum>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QPlainTextEdit" name="resultText">
       <property name="readOnly">
        <bool>true</bool>
       </property>
      </widget>
     </item>
     <item alignment="Qt::AlignHCenter">
      <widget class="QPushButton" name="clearBtn">
       <property name="maximumSize">
        <size>
         <width>75</width>
         <height>16777215</height>
        </size>
       </property>
       <property name="text">
        <string>清除</string>
       </property>
      </widget>
     </item>
    </layout>
   </item>
  </layout>
 </widget>
 <resources/>
 <connections/>
</ui>
```

