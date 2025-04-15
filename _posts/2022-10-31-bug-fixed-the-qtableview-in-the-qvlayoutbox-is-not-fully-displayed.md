---
layout: post
title:  【解决bug】qt的QVLayoutBox中的qtableview显示不全,部分内容隐藏，底下的几行看不到，滚动条的一部分也看不到
date: 2022-10-31
categories:
- c/c++
tags: [qt]
---

![](/images/post/qvlayoutbox-is-not-fully-displayed-1.png)
![](/images/post/qvlayoutbox-is-not-fully-displayed-2.gif)
问题描述：qtableview显示不全，底下的几行看不到，滚动条的一部分也看不到。
```c++
//mainwindow的代码，主界面代码
.............省略代码
this->ui->layout_table_area->addWidget(&tableView);//layout_table_area是一个QVLayoutBox 
.............省略代码
```
```c++
//自定义的mytableview的定义
class MyTableView : public QWidget
{
	Q_OBJECT
public:
	explicit MyTableView(QWidget *parent = nullptr);
	void addRow(QString& tmpl_name);
	void setModel();
 private:
	QTableView  view;
	QStandardItemModel model;
	QMenu menu;
	bool eventFilter(QObject* obj, QEvent *evt);//事件过滤器
signals:
	void delete_item_signal(QString name);
public slots:
	void onDelete(void);
};
```
```c++
//tableView类的相关代码
.....省略代码 
(view.horizontalHeader())->setVisible(false);
(view.verticalHeader())->setVisible(false);//去掉自动序号列 
view.setFocusPolicy(Qt::NoFocus);//取消焦点 
view.setEditTriggers(QAbstractItemView::NoEditTriggers);//设置无法编辑 view.setSelectionMode(QAbstractItemView::SingleSelection);//设置视图只能选择一个项目 view.setSelectionBehavior(QAbstractItemView::SelectRows);//设置视图只能选择行 
view.setModel(&model);//设置显示模型 
view.horizontalHeader()->setSectionResizeMode(QHeaderView::Stretch);//自适应所有列，让它布满空间 
,,,,,省略代码
```
```c++
//tableView中添加行的代码，测试过StandardItem是添加成功的
void MyTableView::addRow(QString& tmpl_name)
{ 
QStandardItem* item = new QStandardItem;
item->setData(tmpl_name, Qt::DisplayRole); 
model.appendRow(item); 
int t = model.rowCount(); 
qDebug()<< QString(t);
}
```
解决方法：
把继承widget，修改为继承QtableView。并且修改界面显示逻辑，不再放入QVLayoutBox。改为控件提升。
class MyTableView : public QTableView
![](/images/post/qvlayoutbox-is-not-fully-displayed-3.png)
这样就可以了。

