---
layout: post
title:  关于风机协议工具题目的解答
date: 2024-12-20
categories:
- python
tags: [pyqt5]
---
架构采用python3.8+pyqt5
先来看下原题：
![](/images/post/fan-protocol-tool-1.png)
视频中软件的效果
![](/images/post/fan-protocol-tool-2.png)
看看解答出的程序效果怎么样？
![](/images/post/fan-protocol-tool-3.gif)
对应代码已经上传到了gitcode
https://gitcode.com/m0_37662818/fan_protocol_tool/overview

实现中的难点是双悬浮可视化，同时要高亮悬浮对应内容，程序中用的的事件过滤器。也许原题解用的是更好的方法。

代码如下
app.py 主文件
```python
from PyQt5 import QtWidgets,QtCore,QtGui
from ui_main import Ui_MainWindow  # 导入主界面类
import sys
import socket
import traceback 
from typing import List, Tuple
      
class MySignal(QtCore.QObject):
    update_sheet = QtCore.pyqtSignal(str)
    hover_signal = QtCore.pyqtSignal(int)
my_signal = MySignal()

class MainWindow(QtWidgets.QMainWindow):
    def __init__(self, parent=None):
        super().__init__()
        self.ui = Ui_MainWindow()
        self.ui.setupUi(self)  # 加载主界面
        self.socket_fan = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 创建socket对象
        self.ui.btn_connect_server.clicked.connect(self.connect_fan)  # 绑定按钮事件
        self.ui.stackedWidget.setCurrentIndex(0)
        self.ui.msg_list.setFlow(QtWidgets.QListView.Flow.LeftToRight)
        self.ui.msg_list.setViewMode(QtWidgets.QListView.ViewMode.IconMode)
        self.ui.msg_list.setResizeMode(QtWidgets.QListView.ResizeMode.Adjust)
        self.ui.msg_list.setWrapping(True)
        self.ui.msg_list.setSpacing(3)

        self.ui.msg_head_list.setFlow(QtWidgets.QListView.Flow.LeftToRight)
        self.ui.msg_head_list.setWrapping(True)
        self.ui.msg_head_list.setViewMode(QtWidgets.QListView.ViewMode.IconMode)
        self.ui.msg_head_list.setResizeMode(QtWidgets.QListView.ResizeMode.Adjust)

        self.ui.msg_head_list.setFixedHeight(70)

        self.ui.msg_list.setDragEnabled(False)
        self.ui.msg_head_list.setDragEnabled(False)
        self.ui.msg_body_table.horizontalHeader().setSectionResizeMode(QtWidgets.QHeaderView.ResizeMode.Stretch)
        self.ui.msg_body_table.verticalHeader().hide()
        self.ui.msg_body_table.horizontalHeader().setStyleSheet("color:#00789d;")

        self.ui.msg_list.setAttribute(QtCore.Qt.WidgetAttribute.WA_Hover,True)
        self.ui.msg_body_table.setAttribute(QtCore.Qt.WidgetAttribute.WA_Hover,True)
        self.ui.msg_head_list.setAttribute(QtCore.Qt.WidgetAttribute.WA_Hover,True)
        
        self.ui.msg_list.installEventFilter(self)
        self.ui.msg_body_table.installEventFilter(self)
        self.ui.msg_head_list.installEventFilter(self)

        self.last_hover_index = -1
        self.msg_id = 0

        self.ui.listWidget.itemClicked.connect(self.list_widget_clicked)
        self.ui.btn_set.clicked.connect(self.btn_set_clicked)
        self.ui.btn_get_sn.clicked.connect(self.btn_get_sn_clicked)
        self.ui.btn_get_state.clicked.connect(self.btn_get_state_clicked)

        my_signal.update_sheet.connect(self.update_sheet_cb)
        my_signal.hover_signal.connect(self.draw_cb)

    def draw_cb(self,index):
        def draw_(index:int,color:str):
            self.ui.msg_list.item(index).setBackground(QtGui.QColor(color))
            if index < 3:
                self.ui.msg_head_list.item(index * 2 + 1).setBackground(QtGui.QColor(color))
            else:
                row = index//3 - 1
                col = index%3 + 1
                self.ui.msg_body_table.item(row,col).setBackground(QtGui.QColor(color))

        if(index == self.last_hover_index):
            return
        if(self.last_hover_index != index):
            if(self.last_hover_index >= 0):
                draw_(self.last_hover_index,'white')
            if(index >= 0):
                draw_(index,'orangered')

        if(index < 0 and self.last_hover_index >= 0):
            draw_(self.last_hover_index,'white')
        self.last_hover_index = index

    def eventFilter(self, watched:QtWidgets.QWidget, event:QtCore.QEvent):
        
        if(watched == self.ui.msg_head_list):
        
            if(event.type() == QtCore.QEvent.Type.HoverMove):
                index = watched.indexAt(event.pos()).row()
                my_signal.hover_signal.emit(index//2)
            if(event.type() == QtCore.QEvent.Type.HoverLeave):
                my_signal.hover_signal.emit(-1)

        if(watched == self.ui.msg_list):
            if(event.type() == QtCore.QEvent.Type.HoverMove):
                index = watched.indexAt(event.pos()).row()
                my_signal.hover_signal.emit(index)
            if(event.type() == QtCore.QEvent.Type.HoverLeave):
                my_signal.hover_signal.emit(-1)
        
        if(watched == self.ui.msg_body_table):
            if(event.type() == QtCore.QEvent.Type.HoverMove):
                x = event.pos().x()
                y = event.pos().y()
                
                hight = watched.horizontalHeader().height()
                point = QtCore.QPoint(x,y - hight)
                Model_index = watched.indexAt(point)
                p = event.pos()

                
                if Model_index is None:
                    my_signal.hover_signal.emit(-1)
                
                else:
                    index = -1
                    row = Model_index.row()
                    col = Model_index.column()
                    if(row >= 0 and col >= 1):
                        row = Model_index.row()
                        #print("row:" + str(row))
                        col = Model_index.column()
                        #print("col:" + str(col))
                        index = (row) * 3 + col - 1 + 3
                    my_signal.hover_signal.emit(index) 
            if(event.type() == QtCore.QEvent.Type.HoverLeave):
                my_signal.hover_signal.emit(-1)

        return super().eventFilter( watched, event)       

    def update_sheet_cb(self,text:str):
        def fill_data(infos:List[Tuple[str,str]]):
            for index,info in enumerate(infos):
                self.ui.msg_list.addItem(info[0])
                if index < 3:
                    if index == 0:
                        self.ui.msg_head_list.addItem("消息长度:")
                    if index == 1: 
                        self.ui.msg_head_list.addItem("类型:")
                    if index == 2:
                        self.ui.msg_head_list.addItem("消息ID:")
                    self.ui.msg_head_list.addItem(str(info[1]))
                elif index % 3 == 0:
                    self.ui.msg_body_table.insertRow(index // 3 - 1)
                    table_item = QtWidgets.QTableWidgetItem(str(infos[index][1]))
                    table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                    self.ui.msg_body_table.setItem(index // 3 - 1, 1, table_item)

                    table_item = QtWidgets.QTableWidgetItem(str(infos[index + 1][1]))
                    table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                    self.ui.msg_body_table.setItem(index // 3 - 1, 2, table_item)

                    table_item = QtWidgets.QTableWidgetItem(str(infos[index + 2][1]))
                    table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                    self.ui.msg_body_table.setItem(index // 3 - 1, 3, table_item)

                    if infos[index][0] == "01":
                        table_item = QtWidgets.QTableWidgetItem("设备编号")
                        table_item.setForeground(QtGui.QColor(255,128,0))
                        table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                        self.ui.msg_body_table.setItem(index // 3 - 1, 0, table_item)
                    elif infos[index][0] == "02":

                        table_item = QtWidgets.QTableWidgetItem("设备状态")
                        table_item.setForeground(QtGui.QColor(255,128,0))
                        table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                        self.ui.msg_body_table.setItem(index // 3 - 1, 0,table_item)
                    elif infos[index][0] == "a0":

                        table_item = QtWidgets.QTableWidgetItem("结果码")
                        table_item.setForeground(QtGui.QColor(255,128,0))
                        table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                        self.ui.msg_body_table.setItem(index // 3 - 1, 0,table_item)
                    elif infos[index][0] == "a1":

                        table_item = QtWidgets.QTableWidgetItem("结果描述")
                        table_item.setForeground(QtGui.QColor(255,128,0))
                        table_item.setTextAlignment(QtCore.Qt.AlignmentFlag.AlignCenter)
                        self.ui.msg_body_table.setItem(index // 3 - 1, 0, table_item)
                    
            #上色
            for i in range(self.ui.msg_list.count()):
                if i < 3:
                    self.ui.msg_list.item(i).setForeground(QtGui.QColor(0,128,0))
                elif i % 3 == 0:
                    self.ui.msg_list.item(i).setForeground(QtGui.QColor(255,128,0))
                elif i % 3 == 1:
                    self.ui.msg_list.item(i).setForeground(QtGui.QColor(0, 120, 157))
                
                
            pass
        
        res = bytes.fromhex(text)
        infos = self.resolve_response(res)
        self.ui.msg_list.clear()
        self.ui.msg_head_list.clear()
        self.ui.msg_body_table.setRowCount(0)
        fill_data(infos)

        pass
    
    def connect_fan(self):
        
        ip = self.ui.input_server_ip.text()
        port = int(self.ui.input_server_port.text())
        print(ip, port)
        if not ip or not port:
            self.ui.msgWindow.append("请输入风机IP和端口！")
            return
        if self.ui.btn_connect_server.text() == "连接":
            try:
                self.socket_fan.connect((ip, port))  # 连接风机
                self.ui.msgWindow.append("连接风机成功！")
                #self.ui.btn_connect_server.setText("断开")
            except Exception as error:
                self.ui.msgWindow.append("连接风机失败！")
                print(error)
        elif self.ui.btn_connect_server.text() == "断开":
            self.socket_fan.close()  # 断开连接
            self.ui.msgWindow.append("断开风机连接！")
            self.ui.btn_connect_server.setText("连接")

    def btn_set_clicked(self):
        text = self.ui.btn_set.text()
        if(text == '设置'):
            self.set_dev_stats_process()

    def btn_get_sn_clicked(self):
        self.get_dev_stats_process(state=1)

    def btn_get_state_clicked(self):
        self.get_dev_stats_process(state=2)


    def list_widget_clicked(self,item:QtWidgets.QListWidgetItem):
        text = item.text()
        if(text == "连接设备"):
            self.change_page_connect()
        elif(text == "设置风泵状态"):
            self.change_page_set()
        elif(text == "读取风泵状态"):
            self.change_page_get()


    def change_page_connect(self):
        self.ui.stackedWidget.setCurrentIndex(0)
    def change_page_set(self):
        self.ui.stackedWidget.setCurrentIndex(1)
    def change_page_get(self):
        self.ui.stackedWidget.setCurrentIndex(2)

    # 获取设备配置请求
    def get_dev_config_request(self,msg_id:int,state:int)-> bytes:
        body = b''
        body += b'\xC1'
        body += b'\x03'
        body += (state.to_bytes(1,byteorder='big'))
        msg_len = 8 + len(body)
        msg_type = 0x01F0
        msg_id = msg_id
        return msg_len.to_bytes(2,byteorder='big') + msg_type.to_bytes(2,byteorder='big') + msg_id.to_bytes(4,byteorder='big') + body

    # 获取设备配置通信过程
    def get_dev_stats_process(self,state=0):
        msg_id = self.msg_id

        try:
            req = self.get_dev_config_request(msg_id, state)
            self.socket_fan.send(req)
            self.ui.msgWindow.append("发送消息：")
            self.ui.msgWindow.append("<font color=\"#006400\">"+ req.hex() + "</font> ")
            res =  self.socket_fan.recv(1024)
            self.ui.msgWindow.append("接收消息:")
            self.ui.msgWindow.append("<font color=\"#006400\">"+ res.hex() + "</font> ")

            infos = self.resolve_response(res)
            for index,info in enumerate(infos):
                if index == 3 or index == 6:
                    if info[0] == "01":
                        self.ui.output_devSn.setText(str(infos[index + 2][1]))
                    elif info[0] == "02":
                        self.ui.output_devState.setText(str(infos[index + 2][1]))
            my_signal.update_sheet.emit(res.hex())
        except Exception as e:
            traceback.print_exc()
        
        self.msg_id += 1
        
        pass

    # 组装设备配置请求
    def set_dev_config_request(self,msgid:int,dev:str, stats:int):
        body = b''

        if dev is not None:
            body += b'\x01'
            body += (len(dev) + 2).to_bytes(1,byteorder='big')
            body += dev.encode()
        if stats is not None:
            body += b'\x02'
            body += (3).to_bytes(1,byteorder='big')
            body += stats.to_bytes(1,byteorder='big')
        msg_len = 8 + len(body)
        msg_type = b'\x02\xF0'
        msgid = msgid
        return msg_len.to_bytes(2,byteorder='big') + msg_type + msgid.to_bytes(4,byteorder='big') + body

    # 设置设备状态通信过程
    def set_dev_stats_process(self):
        msg_id = self.msg_id
        dev_id_text =  self.ui.input_devSn.text() if len(self.ui.input_devSn.text()) > 0 else  None
        dev_stats_text = self.ui.input_devState.text()
        dev_stats = None
        if dev_stats_text is not None and len(dev_stats_text) != 0 :
            dev_stats = int(dev_stats_text)

        try:
            req = self.set_dev_config_request(msgid=msg_id,dev = dev_id_text,stats=dev_stats)
            self.socket_fan.send(req)
            self.ui.msgWindow.append("发送消息：")
            self.ui.msgWindow.append("<font color=\"#006400\">"+ req.hex() + "</font> ")
            res =  self.socket_fan.recv(1024)
            self.ui.msgWindow.append("接收消息:")
            self.ui.msgWindow.append("<font color=\"#006400\">"+ res.hex() + "</font> ")
            my_signal.update_sheet.emit(res.hex())
        except Exception as e:
            traceback.print_exc()
        
        self.msg_id += 1
        pass

    # 解析响应 bytes版
    '''
    def resolve_response(self, res:bytes)-> dict:
        if len(res) < 8:
            return {}
        infos = {}
        msg_type_bytes = res[2:4]
        msg_type = 'unknown massage '

        if msg_type_bytes == b'\x01\xF1':
            msg_type = 'read state response'
        elif msg_type_bytes == b'\x02\xF1':
            msg_type = 'set state response'

        infos["msg_len"] = int.from_bytes(res[0:2],byteorder='big') 
        infos["msg_id"] =  int.from_bytes(res[4:8],byteorder='big')
        infos["msg_type"] = msg_type
        pos = 8
        while pos < len(res):
            code = res[pos:pos + 1]
            if(msg_type_bytes == b'\x01\xF1'):
                
                if code == b'\x01':
                    length = res[pos + 1]
                    desc = res[pos+2:pos+length].decode()
                    field = Field(b'\x01',length,desc)
                    infos['sn'] = field
                    pos += length
                elif code == b'\x02':
                    length = res[pos + 1]
                    desc = int.from_bytes(res[pos + 2:pos + length],byteorder='big')
                    field = Field(b'\x02',length,desc)
                    infos['state'] = field
                    pos += length
                else:
                    break
            if(msg_type_bytes == b'\x02\xF1'):
                if code == b'\xA0':
                    length = res[pos + 1]
                    desc = int.from_bytes[pos + 2: pos + length]
                    field = Field(b'\xA0',length,desc)
                    infos['code'] = field
                    pos += length
                elif code == b'\xA1':
                    length = res[pos + 1]
                    desc = res[pos+2:pos + length].decode()
                    field = Field(b'\xA1',length,desc)
                    infos['desc'] = field
                    pos += length
                else:
                    break
        return infos

        pass
    '''

    # 解析响应 str版
    def resolve_response(self, res:bytes)-> list:
        def resolve_response_head(res_head:bytes)->list:
            infos = []
            msg_type = 0
            msg_len_hex = (res_head[0:2]).hex()
            msg_type_hex = (res_head[2:4]).hex()
            msg_id_hex = (res_head[4:8]).hex()
            msg_type_desc = ""

            msg_len = int.from_bytes(res_head[0:2],byteorder='big')
            msg_id = int.from_bytes(res_head[4:8],byteorder='big')

            if(msg_type_hex.upper() == "01F1"):
                    msg_type_desc = "get_state_response"
            elif(msg_type_hex.upper() == "02F1"):
                    msg_type_desc = "set_state_response"

            infos.append((msg_len_hex,msg_len))
            infos.append((msg_type_hex,msg_type_desc))
            infos.append((msg_id_hex,msg_id))
            return infos

        def resolve_response_body_get_state(res_body:bytes)->list:
            pos = 0
            infos = []
            while pos < len(res_body):
                code_hex = res_body[pos:pos + 1].hex()
                length_hex = res_body[pos + 1:pos + 2].hex()
                length = res_body[1]
                value_hex = res_body[pos + 2: pos + length].hex()
                value = ""
                if(code_hex == "01"):
                    value = res_body[pos + 2:pos + length].decode(encoding="ANSI")
                elif(code_hex == "02"):
                    value = int.from_bytes(res_body[pos + 2:pos + length],byteorder='big')
                infos.append((code_hex,code_hex))
                infos.append((length_hex,str(length)))
                infos.append((value_hex,str(value)))
                pos += length
            return infos

        def resolve_response_body_set_state(res_body:bytes)->list:
            pos = 0
            infos = []
            while pos < len(res_body):
                code_hex = res_body[pos:pos + 1].hex()
                length_hex = res_body[pos + 1:pos + 2].hex()
                length = res_body[pos+1]
                value_hex = res_body[pos + 2:pos + length].hex()
                value = ""
                if(code_hex.upper() == "A0"):
                    value = res_body[pos+2]
                elif(code_hex.upper() == "A1"):
                    value = res_body[pos + 2:pos + length].decode(encoding="ANSI")
                infos.append((code_hex,code_hex))
                infos.append((length_hex,str(length)))  
                infos.append((value_hex,str(value)))
                pos += length
            return infos

        res_len = len(res)
        if res_len < 8:
            return []
        msg_len = int.from_bytes(res[0:2],byteorder='big')
        if msg_len > res_len:
            return []
        res_head = res[0:8]
        res_body = res[8:msg_len]
        infos = []
        infos = resolve_response_head(res_head)
        if(infos[1][0].upper() == "01F1"):
            infos = infos + resolve_response_body_get_state(res_body)
        elif(infos[1][0].upper() == "02F1"):
            infos = infos + resolve_response_body_set_state(res_body)
        return infos

if __name__ == '__main__':
    app = QtWidgets.QApplication([])  # 创建QApplication对象
    window = MainWindow()  # 创建主界面对象
    window.show()  # 显示主界面
    sys.exit(app.exec_())  # 运行主界面

```