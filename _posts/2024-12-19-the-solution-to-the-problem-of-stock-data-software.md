---
layout: post
title:  关于仿写股票数据软件题目的解答
date: 2024-12-19
categories:
- python
tags: [pyside]
---

**原题**
![](/images/post/stock-data-software-1.png)
**对应问题视频截图**
![](/images/post/stock-data-software-2.png)
**实现的效果**
![](/images/post/stock-data-software-3.gif)

**不同点**

实现的作品和原题要求的不同点

1. 题目要求爬虫获取历史数据，作品中是调库获取所有股票历史数据
2. 实时数据使用爬虫的方式爬取指定股票的数据，需要实时更新，我做了修改，改成按下按钮获取分时数据，不自动刷新

**体会**
1. 先AKShare获取数据，总是被封IP，后改用证券宝
2. 股票当天分时数据不好获得，这个数据是由服务器主动推送过来的，通过监听所有的收发包数据，找到了这个规律才获得数据

代码在gitcode地址：https://gitcode.com/m0_37662818/stock_quotation/blob/main/README.md?init=initTree

**解答**：
```python
# python版本要3.9以上akshare才可以获取所有股票数据
# 去掉AKShare改用证券宝接口


import pyqtgraph as pg
import baostock as bs
import pandas as pd
import re
import numpy as np
import json
import traceback
from threading import  Thread
from PySide6 import QtWidgets,QtCore,QtGui
from datetime import datetime
from time import sleep
from selenium import webdriver
from selenium.webdriver import DesiredCapabilities
from selenium.webdriver.chrome.options import Options
from datetime import datetime

#数据源接口
class StockData:
    @staticmethod
    def get_all_stock_code() -> pd.DataFrame:
        # 证券宝接口
        lg = bs.login()
        # 显示登陆返回信息
        #print('login respond error_code:'+lg.error_code)
        #print('login respond  error_msg:'+lg.error_msg)

        yesterday= (datetime.now() - pd.Timedelta(days=1)).strftime('%Y-%m-%d')
        three_months_ago = (datetime.now() - pd.Timedelta(days=90)).strftime('%Y-%m-%d')
        #### 获取交易日信息 ####
        rs = bs.query_trade_dates(start_date=three_months_ago, end_date=yesterday)
        #print('query_trade_dates respond error_code:'+rs.error_code)
        #print('query_trade_dates respond  error_msg:'+rs.error_msg)
        
        if rs.error_code!= '0':
            bs.logout()
            return None

        data_list = []
        while (rs.error_code == '0') & rs.next():
        # 获取一条记录，将记录合并在一起
            data_list.append(rs.get_row_data())
        result = pd.DataFrame(data_list, columns=rs.fields)
        #从后面开始查找，找到一行is_trading_day=1的日期
        trading_days = result[result['is_trading_day']=='1']
        if trading_days.empty:
            print('没有交易日数据')
            bs.logout()
            return None
        
        last_trading_day = trading_days.iloc[-1]['calendar_date']
        #print(str(last_trading_day) + str(type(last_trading_day)))

        #获取所有股票代码

        #### 获取某日所有证券信息 ####
        rs = bs.query_all_stock(day=str(last_trading_day))
        #print('query_all_stock respond error_code:'+rs.error_code)
        #print('query_all_stock respond  error_msg:'+rs.error_msg)

        #### 打印结果集 ####
        data_list = []
        while (rs.error_code == '0') & rs.next():
            # 获取一条记录，将记录合并在一起
            data_list.append(rs.get_row_data())
        result = pd.DataFrame(data_list, columns=rs.fields)

        #只保留sz.00开头的，sh.60开头的沪深正股数据
        result = result[(result['code'].str.startswith('sz.00')) | (result['code'].str.startswith('sh.60'))]

    
        #去掉index列
        #result.drop(columns=['index'], inplace=True)

        #panda复制code列，重命名为bs_code
        result['bs_code'] = result['code']
        #重命名code_name 为name
        result.rename(columns={'code_name':'name'}, inplace=True)
        #将code列前面的sz.或者sh.去掉
        result['code'] = result['code'].str.replace('sz.', '')
        result['code'] = result['code'].str.replace('sh.', '')

        #排序
        result['sort'] = result['code'].astype(int)
        result.sort_values(by='sort', inplace=True)
        result.drop(columns=['sort'], inplace=True)
        #重置索引
        result.reset_index(drop=True, inplace=True)

        #print(result)
        #### 登出系统 ####
        bs.logout()
        return result
    
    @staticmethod
    def get_history_data(code:str, start_date:str, end_date:str) -> pd.DataFrame:
        # 证券宝接口
        lg = bs.login()

        if lg.error_code != '0':
            print('证券宝登录失败')
            return None
        
        #### 获取历史行情 ####
        rs = bs.query_history_k_data_plus(code,
        "date,code,open,close",
        start_date=start_date, end_date=end_date, 
        frequency="d", adjustflag="2") #frequency="d"取日k线，adjustflag="3"默认不复权
        #print('query_history_k_data_plus respond error_code:'+rs.error_code)
        #print('query_history_k_data_plus respond  error_msg:'+rs.error_msg)
        #### 打印结果集 ####
        data_list = []
        while (rs.error_code == '0') & rs.next():
            # 获取一条记录，将记录合并在一起
            data_list.append(rs.get_row_data())
        result = pd.DataFrame(data_list, columns=rs.fields)
        if(result.empty):
            my_signal.warning_signal.emit('没有查询到数据')
            bs.logout()
            return None
        
        #修改列类型
        result['open'] = result['open'].astype(float)
        result['close'] = result['close'].astype(float)

        #### 登出系统 ####
        bs.logout()
        return result

    @staticmethod
    def get_realtime_data(code:str) -> pd.DataFrame:
        # 先去掉所有非数字的字符，留下来六位数字str，对code进行处理，如果以60开头，添加sh前缀，如果以00开头，添加sz前缀
        code = ''.join(char for char in code if char.isdigit())
        if len(code)!= 6:
            return None

        if code.startswith('60'):
            code = 'sh' + code
        elif code.startswith('00'):
            code ='sz' + code

        print(code)
        options = Options()
        result_df = pd.DataFrame(columns=['time','price'])
        caps = {

            "browserName": "chrome",
            'goog:loggingPrefs': {'performance': 'ALL'}  # 开启日志性能监听
        }

        # 将caps添加到options中
        for key, value in caps.items():
            options.set_capability(key, value)

        options.add_argument('--headless')  # 无头模式

        # 启动浏览器
        driver = webdriver.Chrome(options=options)
        url = f"https://quote.eastmoney.com/concept/{code}.html"
        print(url)
        driver.get(url)
        sleep(3)  # wait for the requests to take place

        # extract requests from logs
        logs_raw = driver.get_log("performance")

        logs = [json.loads(lr["message"])["message"] for lr in logs_raw]

        def log_filter(log_):
            return (
                # is an actual response
                log_["method"] == "Network.eventSourceMessageReceived"
            )

        for log in filter(log_filter, logs):
            try:

                #判断log中params是否存在,且params中是否存在data
                if 'params' not in log or 'data' not in log['params']:
                    continue

                data = json.loads(log['params']['data'])
                #先判断data是否存在,且data->trends是否存在
                if 'data' not in data or 'trends' not in data['data']:
                    continue

                data = data['data']['trends']
                #判断data是否是list
                if isinstance(data, list):
                    for item in data:
                        arr =item.split(',')
                        t = datetime.strptime(str(arr[0]),"%Y-%m-%d %H:%M")
                        #判断时间是否在9:30-11:30 13:00-15:00之间
                        if (t.hour <= 9 and t.minute < 30):
                            continue

                        if(t.hour >= 15 and t.minute > 0):
                            continue
                            
                        if(t.hour == 12):
                            continue

                        if(t.hour == 11 and t.minute > 30):
                            continue

                        #双重保险
                        if (t.hour >= 9 and t.minute >= 30) or (t.hour >= 10) :
                            avg = round((float(arr[1]) + float(arr[2]))/2,2)
                            new_row = pd.DataFrame([[t,avg]],columns=['time','price'],index=[0])
                            if result_df.empty:
                                result_df = new_row
                            else:
                                result_df = pd.concat([result_df, new_row],ignore_index=True)

            except Exception  as e:
                print(log)
                print(traceback.print_exc())
                pass
        driver.quit()
        #print(result_df)
        if result_df.empty:
            return None

        return result_df
        
        
#信号库
class MySignal(QtCore.QObject):
    combox_update_signal = QtCore.Signal()
    history_area_data_signal = QtCore.Signal()
    realtime_area_data_signal = QtCore.Signal()
    warning_signal = QtCore.Signal(str)
#    start_timer_signal = QtCore.Signal()
#    Stop_timer_signal = QtCore.Signal()
#信号的实例化
my_signal = MySignal()

#主界面
class MainWindow(QtWidgets.QWidget):
    def __init__(self):
        super().__init__()

        # 数据，用panda处理
        self.stock_code_df : pd.DataFrame  = None #股票代号数据
        self.history_data_df : pd.DataFrame = None #历史行情数据
        self.realtime_data_df : pd.DataFrame = None #实时行情数据
        self.today_data_df : pd.DataFrame = None #今日行情数据

        # 界面布局
        self.setWindowTitle('股票行情查看软件')

        self.combox = QtWidgets.QComboBox(self)
        self.search_btn = QtWidgets.QPushButton('同步股票清单', self)

        self.is_searching_stock_code = False

        self.search_edit = QtWidgets.QLineEdit(self)
        self.search_edit.setStyleSheet("QLineEdit { border: 1px solid #888888; }")
        self.tab_widget = QtWidgets.QTabWidget(self)
        self.history_page = QtWidgets.QWidget(self)
        self.realtime_page = QtWidgets.QWidget(self)
        self.tab_widget.addTab(self.history_page, '历史行情')
        self.tab_widget.addTab(self.realtime_page, '实时行情')
        
        self.layout = QtWidgets.QVBoxLayout(self)

        # 头部，查询股票代码布局
        self.header_layout = QtWidgets.QHBoxLayout()

        self.header_layout.addSpacing(20)
        self.header_layout.addWidget(self.search_btn)
        self.header_layout.setStretchFactor(self.search_btn,1)
        self.header_layout.addSpacing(20)
        self.search_label = QtWidgets.QLabel('请输入股票名称或代码')
        self.search_label.setAlignment(QtCore.Qt.AlignCenter)
        self.header_layout.addWidget(self.search_label)
        self.header_layout.setStretchFactor(self.search_label,0)
        self.header_layout.addSpacing(20)
        self.header_layout.addWidget(self.search_edit)
        self.header_layout.addSpacing(1)
        self.header_layout.addWidget(self.combox)
        self.header_layout.setStretchFactor(self.search_edit,2)
        self.header_layout.setStretchFactor(self.combox,3)

        self.search_edit.textChanged.connect(self.search_edit_cb)

        self.layout.addLayout(self.header_layout)
        self.layout.addWidget(self.tab_widget)
        self.text_label = QtWidgets.QLabel(self)
        self.layout.addWidget(self.text_label)

        # 历史行情页面布局
        self.history_page_layout = QtWidgets.QVBoxLayout(self.history_page)
        self.history_layout = QtWidgets.QHBoxLayout()
        self.history_page_layout.addLayout(self.history_layout)
        self.edit_area = QtWidgets.QTextEdit(self)
        self.history_page_layout.addWidget(self.edit_area)

        
        self.history_plot = pg.PlotWidget()
        self.history_page_layout.addWidget(self.history_plot)



        self.history_page_layout.setStretchFactor(self.edit_area,1)
        self.history_page_layout.setStretchFactor(self.history_plot,2)

        self.history_btn = QtWidgets.QPushButton('历史行情', self)
        self.date_edit_start = QtWidgets.QDateEdit(self)
        self.date_edit_start.setCalendarPopup(True)
        self.date_edit_start.setDisplayFormat("yyyy-MM-dd")
        self.date_edit_start.setAlignment(QtCore.Qt.AlignCenter)
        self.date_edit_start.setDate(QtCore.QDate.currentDate().addYears(-3))

        self.date_edit_end = QtWidgets.QDateEdit(self)
        self.date_edit_end.setCalendarPopup(True)
        self.date_edit_end.setDisplayFormat("yyyy-MM-dd")
        self.date_edit_end.setAlignment(QtCore.Qt.AlignCenter)
        self.date_edit_end.setDate(QtCore.QDate.currentDate())

        self.history_layout.addWidget(QtWidgets.QLabel('时间范围'),1,QtCore.Qt.AlignLeft)
        self.history_layout.addWidget(self.date_edit_start,2)
        self.history_layout.addWidget(QtWidgets.QLabel(' - '),0,QtCore.Qt.AlignCenter)
        self.history_layout.addWidget(self.date_edit_end,2)
        self.history_layout.addWidget(self.history_btn,1,QtCore.Qt.AlignRight)

        self.history_curve = None # 历史行情图的曲线
        self.history_vertical_line = None # 历史行情图的垂直线
        self.history_horizontal_line = None #历史行情图的水平线
       
        self.is_searching_history_data = False


        # 实时行情页面布局
        self.realtime_btn = QtWidgets.QPushButton('实时行情', self)
        self.realtime_page_layout = QtWidgets.QVBoxLayout(self.realtime_page)
        self.realtime_page_layout.addWidget(self.realtime_btn)
        self.realtime_plot = pg.PlotWidget(self)
        self.realtime_page_layout.addWidget(self.realtime_plot)

        self.realtime_curve = None # 实时行情图的曲线
        self.realtime_vertical_line = None # 实时行情图的垂直线
        self.realtime_horizontal_line = None #实时行情图的水平线

        self.is_searching_realtime_data = False

        # 初始化数据
        worker = Thread(target=self.init_data_thread_func)
        worker.start()

        # 信号连接
        my_signal.combox_update_signal.connect(self.combox_update_cb)
        my_signal.history_area_data_signal.connect(self.history_data_cb)
        my_signal.realtime_area_data_signal.connect(self.realtime_data_cb)
        my_signal.warning_signal.connect(self.warning_cb)
        self.history_plot.scene().sigMouseMoved.connect(self.history_plot_mouse_move_cb)
        self.realtime_plot.scene().sigMouseMoved.connect(self.realtime_plot_mouse_move_cb)
        self.history_btn.clicked.connect(self.history_btn_cb)
        self.realtime_btn.clicked.connect(self.realtime_btn_cb)
        self.search_btn.clicked.connect(self.search_btn_cb)
        self.tab_widget.tabBarClicked.connect(self.tab_widget_cb)

        #   定时器 不使用
        #self.realtime_timer = QtCore.QTimer()
        #self.realtime_timer.timeout.connect(self.timer_timeout_cb)
    # 数据加载另外起个线程
    def init_data_thread_func(self):
        # 加载本地股票代号数据,数据为CSV格式,使用panda读取
        # 如果文件不存在,则跳过
        try:
            self.stock_code_df = pd.read_csv('stock_symbol.csv',encoding='utf-8',dtype={'code':str,'tradeStatus':str})
            # 启动初始化加载到combox
            self.add_item_to_combox(self.stock_code_df) 

            #这段代码是临时数据TODO 加载本地历史行情数据,数据为CSV格式,使用panda读取,该数据为测试数据
            #self.history_data_df = pd.read_csv('temp_history_data.csv',encoding='utf-8')
            #self.history_data_df['日期'] = pd.to_datetime(self.history_data_df['日期'],format='%Y-%m-%d')

            # 临时加载实时数据
            #self.realtime_data_df = pd.read_csv('temp_realtime_data.csv',encoding='utf-8')
            #self.realtime_data_df['时间'] = pd.to_datetime(self.realtime_data_df['时间'],format='%Y-%m-%d %H:%M:%S')

        except FileNotFoundError:
            print('股票代号数据文件不存在')
            pass
        except Exception as e:
            print('其他错误' + str(e))
            pass


    def add_item_to_combox(self, df : pd.DataFrame):
        self.combox.clear()
        temp = None
        #加载df前20个数据到combox,如果不到20个则加载全部
        if df.shape[0] > 20:
            temp = df.head(20)
        else:
            temp = df
        for index, row in temp.iterrows():
            self.combox.addItem(row['code'] + "(" + row['name'] + ")")
        my_signal.combox_update_signal.emit()

    def combox_update_cb(self):
        self.combox.setCurrentIndex(0)

    # 查询历史行情的btn回调
    def history_btn_cb(self):
        def history_thread_func():
            text = self.combox.currentText()
            match = re.search(r'\d{6}', text)
            start_time_str = self.date_edit_start.date().toString('yyyy-MM-dd')
            end_time_str = self.date_edit_end.date().toString('yyyy-MM-dd')
       
            if match:
                code = str(match.group())
                #翻译为证券宝格式
                temp = self.stock_code_df[self.stock_code_df['code'] == code]
                if temp is None or temp.shape[0] == 0:
                    my_signal.warning_signal.emit('股票代码错误')
                    self.is_searching_history_data = False
                
                bs_code = temp.iloc[0]['bs_code']
                self.history_data_df = StockData.get_history_data(code=bs_code, start_date=start_time_str, end_date=end_time_str)
                if self.history_data_df is not None:
                    my_signal.history_area_data_signal.emit()

            self.is_searching_history_data = False
            return

        if self.is_searching_history_data:
            return
        self.is_searching_history_data = True
        worker = Thread(target=history_thread_func)
        worker.start()
        return

    # 历史行情图绘制
    def history_data_cb(self):
        self.edit_area.setText(self.history_data_df.to_string())
        history_data_df = self.history_data_df.copy()
        #打印列名和类型
        #print(history_data_df.columns)
        

        history_data_df['avg'] = ((history_data_df['open'] + history_data_df['close'])/2).round(2)
        
        # 行号的序号
        xdata = history_data_df.index.astype(int).to_list()
        ydata = history_data_df['avg'].astype(float).to_list()


        self.history_plot.plotItem.setLabel('left', '均价')
        axis: pg.AxisItem = self.history_plot.getPlotItem().getAxis('bottom')
        
        self.history_plot.plotItem.setLabel('bottom', '日期',units='')
                
        # x轴刻度变为字符串,可以实现日期的显示 这里费了好多脑子
        # self.history_plot.getPlotItem().getAxis('bottom').setTicks([[(x, history_data_df.iloc[index]['日期']) #for #index, x  in enumerate(xdata)]])
        x_ticks_str = [history_data_df.iloc[index]['date'] for index, x in enumerate(xdata)] 
        x_ticks1 = []
        x_ticks2 = []
        x_ticks3 = []


        gap =  (round(len(x_ticks_str)) // 8 + 1 ) # 0 到 7 是 1 ，8到15是 2
        # 设定 gap为 4 和 2 的倍数 
        gap = 2 if gap == 3 else gap
        if gap > 4: # 4 8 12 16 20......
            gap = (gap + 3) // 4 * 4 


        x_ticks1 = [(i,x_ticks_str[i])for i in range(0,len(x_ticks_str),gap)]
        if gap >= 2:
            x_ticks2 = [(i,x_ticks_str[i])for i in range(0,len(x_ticks_str),gap // 2)]
        if gap >= 4 :
            x_ticks3 = [(i,x_ticks_str[i])for i in range(0,len(x_ticks_str),gap // 4)]
        #去掉x_ticks2奇数个元素，不去掉也行
        x_ticks2 = x_ticks2[1::2]
        x_ticks3 = x_ticks3[1::2]
        #x_ticks3 = [(i,x_ticks_str[i])for i in range(0,len(x_ticks_str),gap // 4)]
        plot_ticks = [x_ticks1,x_ticks2,x_ticks3]

        axis.setTicks(plot_ticks)

        #限制显示范围
        

        self.history_plot.plotItem.vb.setLimits(
            xMin = xdata[0],xMax =xdata[-1] *1.2 ,yMin = min(ydata) -(max(ydata) - min(ydata)) * 0.2 , yMax = max(ydata) +(max(ydata) - min(ydata)) *0.2,minXRange=10, maxXRange= (xdata[-1] - xdata[0]) *1.2,minYRange = 0.1, maxYRange = 1.4*(max(ydata) - min(ydata)))
        
        #清除旧的绘制新的
        if self.history_curve is not None:
            self.history_plot.removeItem(self.history_curve)
        self.history_curve = self.history_plot.plotItem.plot(xdata, ydata, pen='w')
        


    #历史行情图的鼠标移动事件
    def history_plot_mouse_move_cb(self,pos):
        if self.history_data_df is None:
            return

        view_coords = pg.Point(pos)
        mouse_point  = self.history_plot.getPlotItem().vb.mapSceneToView(view_coords)
        x = round(mouse_point.x())
        #保留两位小数
        temp_df = self.history_data_df.copy()
        temp_df['avg'] = ((temp_df['open'] + temp_df['close'])/2).round(2)

        temp_df['temp_index'] = temp_df.index.astype(int)
        closest_min_idx = (temp_df['temp_index'] - x).abs().idxmin()
        

        y_value = temp_df.iloc[closest_min_idx]['avg']

        #如果历史行情图还没有绘制,则不绘制垂直线,水平线
        if self.history_curve is not None:
            self.history_plot.removeItem(self.history_vertical_line)
            self.history_plot.removeItem(self.history_horizontal_line)
            self.text_label.setText("价格：" + str(y_value.round(2)) + " 日期：" + temp_df.iloc[closest_min_idx]['date'])


            self.history_vertical_line = pg.InfiniteLine(pos=closest_min_idx, angle=90, pen='y')
            self.history_horizontal_line = pg.InfiniteLine(pos=y_value, angle=0, pen='y')
            self.history_plot.addItem(self.history_horizontal_line)
            self.history_plot.addItem(self.history_vertical_line)

        #print(idx,y_value)

    # 查看实时行情图btn回调
    def realtime_btn_cb(self):
        def reset_():
            self.is_searching_realtime_data = False
            self.realtime_btn.setEnabled(True)

        def realtime_thread_func():
            text = self.combox.currentText()
            match = re.search(r'\d{6}', text)
            
            if match:
                today = QtCore.QDate.currentDate()
                today_str = today.toString('yyyy-MM-dd')
                code = str(match.group())
                
                realtime_data_df = StockData.get_realtime_data(code=code)
               
                #如果获取的价格数据为空,则不更新数据
                if realtime_data_df is None or realtime_data_df.empty:
                    reset_()
                    return
                # 如果获得价格数据为0，则不更新数据
                if realtime_data_df['price'].sum() == 0:
                    reset_()
                    return
                self.realtime_data_df = realtime_data_df
                
                #my_signal.start_timer_signal.emit()
                my_signal.realtime_area_data_signal.emit()
                reset_()

        if self.is_searching_realtime_data:
            return
        self.is_searching_realtime_data = True
        self.realtime_btn.setEnabled(False)
        worker = Thread(target=realtime_thread_func)
        worker.start()
        return
        

    # 实时行情图绘制
    def realtime_data_cb(self):
        #清除旧的绘制新的
        if self.realtime_curve is not None:
            self.realtime_plot.removeItem(self.realtime_curve)

        realtime_data_df = self.realtime_data_df.copy()
        
        # 行号的序号
        xdata = realtime_data_df.index.astype(int).to_list()
        ydata = realtime_data_df['price'].astype(float).to_list()

        self.realtime_plot.plotItem.setLabel('left', '价格')
        axis: pg.AxisItem = self.realtime_plot.getPlotItem().getAxis('bottom')
        
        self.realtime_plot.plotItem.setLabel('bottom', '时间',units='')
                
        # x轴刻度变为字符串,固定显示9:30 10:30 14:00 15:00
        x_ticks_str = [(0,'9:30'),(30,'10:00'),(60,'10:30'),(90,'11:00'),(120,'11:30/13:00'),(150,'13:30'),(180,'14:00'),(210,'14:30'),(240,'15:00')]
        
        plot_ticks = [x_ticks_str]

        axis.setTicks(plot_ticks)

        #限制显示范围

        self.realtime_plot.plotItem.vb.setLimits(
            xMin = 0,xMax =250 ,yMin = min(ydata) - 1 , yMax = max(ydata) +1,minXRange=250, maxXRange= 250,minYRange = 0.1, maxYRange = 1.2*max(ydata))
        

        self.realtime_curve = self.realtime_plot.plotItem.plot(xdata, ydata, pen='w')


        return

    def realtime_plot_mouse_move_cb(self,pos):
        if self.realtime_data_df is None:
            return
        #清除旧线
        self.realtime_plot.removeItem(self.realtime_vertical_line)
        self.realtime_plot.removeItem(self.realtime_horizontal_line)
        closest_min_idx = None

        view_coords = pg.Point(pos)
        mouse_point  = self.realtime_plot.getPlotItem().vb.mapSceneToView(view_coords)
        x = round(mouse_point.x())
        #保留两位小数
        temp_df = self.realtime_data_df.copy()
        
        temp_df['序号'] = temp_df.index.astype(int)
        
        if x < 0 or x > temp_df.iloc[-1]['序号'] or x < temp_df.iloc[0]['序号']:
            return 

        # 查找可以划线的x坐标
        closest_min_idx = (temp_df['序号'] - x).abs().idxmin()
        

        
        #如果实时行情图还没有绘制,则不绘制垂直线,水平线
        if self.realtime_curve is not None and closest_min_idx is not None:
            
            y_value = temp_df.iloc[closest_min_idx]['price']
            self.text_label.setText("价格：" + str(y_value.round(2)) + " 时间：" + temp_df.iloc[closest_min_idx]['time'].strftime('%H:%M:%S'))

            self.realtime_vertical_line = pg.InfiniteLine(pos=closest_min_idx, angle=90, pen='y')
            self.realtime_horizontal_line = pg.InfiniteLine(pos=y_value, angle=0, pen='y')
            self.realtime_plot.addItem(self.realtime_horizontal_line)
            self.realtime_plot.addItem(self.realtime_vertical_line)


    def search_edit_cb(self):
        text = self.search_edit.text()
        if text == '':
            self.add_item_to_combox(self.stock_code_df)
            return
        
        df = self.stock_code_df[self.stock_code_df['name'].str.contains(text) | self.stock_code_df['code'].str.contains(text)]
        self.add_item_to_combox(df)
        return

    # 查询所有股票代号数据的回调
    def search_btn_cb(self):
        # QtWidgets.QApplication.setOverrideCursor(QtCore.Qt.WaitCursor)
        def search_thread_func():
            self.is_searching_stock_code = True
            stock_code_df = StockData.get_all_stock_code()
            #print(stock_code_df)
            if stock_code_df is None:
                print('股票数据获取失败')
            if stock_code_df.equals(self.stock_code_df):
                print('股票数据无更新')
            else:
              
                self.stock_code_df = stock_code_df
                self.stock_code_df.to_csv('stock_symbol.csv',index=False,encoding='utf-8')
                print('股票数据更新成功')

            self.search_btn.setEnabled(True)
            self.is_searching_stock_code = False


        if self.is_searching_stock_code:
            return
        self.is_searching_stock_code = True
        self.search_btn.setEnabled(False)
        #self.search_btn.setEnabled(False)
        worker = Thread(target=search_thread_func)
        worker.start()

    def warning_cb(self,msg):
        QtWidgets.QMessageBox.warning(self, "警告", msg)

    # 切换tab页时清除text_label
    def tab_widget_cb(self,index):
        self.text_label.setText("")

    #def start_timer_cb(self):
    #    self.realtime_timer.start(1000*60 * 5) #5分钟更新一次数据
    
    #def stop_timer_cb(self):
    #    self.realtime_timer.stop()

    #def timer_timeout_cb(self):
        #print("定时器触发" + str(QtCore.QDateTime.currentDateTime().toString('HH:mm:ss')))
        #self.realtime_data_cb()

if __name__ == '__main__':
    app = QtWidgets.QApplication()
    window = MainWindow()
    window.show()
    app.exec()

```