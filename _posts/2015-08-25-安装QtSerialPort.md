---
layout: post
title:  "安装 QtSerialPort"
date:   2015-08-25 00:11:26 +0800
categories: [Embedded, Qt]
---

## 1. 编译安装

```
git clone git://gitorious.org/qt/qtserialport.git
cd qtserialport
qmake qtserialport.pro
make
sudo make install
```

QtSerialPort库将会被安装至Qt的lib目录下，例如：

```
/usr/local/Trolltech/Qt-4.8.6/lib/libQtSerialPort.so.1.0
```

## 2. 使用

在 *.pro 文件中添加下列内容：

```
CONFIG += serialport
```

## 注意

为嵌入式版本安装时，采用qtcreator的安装方式安装，否则依旧会被安装在桌面版的目录下。

 1. download and unpack the QtSerialPort sources
 2. run QtCreator and open the “qtserialport.pro” project file
 3. get to “Projects->(Your Kit)->Build->Build Steps”
 4. add a new make “Build Step” and write to the “Make arguments” the install target
 5. from the menus, select “Rebuild Project qtserialport”