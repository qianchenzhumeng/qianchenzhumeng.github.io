---
layout: post
title:  "Qt/Embeded(4.8.6)交叉编译环境搭建"
date:   2015-08-25 00:11:26 +0000
categories: [Embedded, Qt]
---

目标板为树莓派，安装镜像为：2014-12-24-wheezy-raspbian.img

## 安装交叉编译器：

下载：[gcc-linaro-arm-linux-gnueabihf-raspbian-x64](https://github.com/raspberrypi/tools/archive/master.zip)

解压后将bin目录添加进~/.profile文件中，例如：

```
 PATH=/home/qianchen/rpi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin:$PATH
 export PATH
```

运行以下命令：

```
 source ~/.profile
```

### 在Qt Creator中添加交叉编译器

工具->选项->编译器，点击“添加”按钮，选择“GCC”，“名称”一栏输入“ARM GCC”(可自定义)，“编译器路径”一栏选择/home/qianchen/rpi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/binarm-linux-gnueabihf-g++，单击“确定”按钮，添加成功。


## 安装qt/embedded

注意：如果需要添加触摸屏支持（tslib），相关设置要在make（make过程会花费数小时，取决于机器配置）之前完成，请跳转至最后：qt/embedded添加触摸屏支持。

下载：[qt-everywhere-opensource-src-4.8.6.tar.gz](http://download.qt-project.org/archive/qt/4.8/4.8.6/)

```
tar -xvzf qt-everywhere-opensource-src-4.8.6.tar.gz 
cd qt-everywhere-opensource-src-4.8.6
```

修改qmake.conf文件，选择之前安装的交叉编译编译工具链（即将相应工具替换为arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin下的工具:）：

```
vim mkspecs/qws/linux-armv6-g++/qmake.conf
```

vim命令模式下输入：

```
%s/arm-linux/arm-linux-gnueabihf/g
```

即可完成替换。

```
 ./configure -v -opensource -confirm-license -embedded arm  -xplatform qws/linux-armv6-g++
 make
 sudo make install
```

程序将会安装在/usr/local/Trolltech/QtEmbedded-4.8.6-arm文件夹下。

## 为QtEmbedded-4.8.6-arm添加qwt-6.1.0

打开qwt源码包中的qwtconfig.pri文件,找到：
```
QWT_INSTALL_PREFIX    = /usr/local/qwt-$$QWT_VERSION
```
将其修改为：
```
QWT_INSTALL_PREFIX    = /usr/local/Trolltech/qwt-$$QWT_VERSION-arm
```
找到：
```
QWT_CONFIG     += QwtOpenGL
QWT_CONFIG     += QwtDesigner
```
将其注释掉：
```
\#QWT_CONFIG     += QwtOpenGL
\#QWT_CONFIG     += QwtDesigner
```
逐条运行下列命令：

```
/usr/local/Trolltech/QtEmbedded-4.8.6-arm/bin/qmake
make
sudo make install
sudo cp /usr/local/Trolltech/qwt-6.1.0-arm/lib/* /usr/local/Trolltech/QtEmbedded-4.8.6-arm/lib/
```

需要用到qwt时，在*.pro文件中添加以下两行：
```
LIBS += -lqwt
INCLUDEPATH += /usr/local/Trolltech/qwt-6.1.0-Embedded-arm/include
```
之后将/usr/local/Trolltech/QtEmbedded-4.8.6-arm复制到目标板（树莓派）上(只复制lib目录的话无法加载sqlite驱动)，方便起见，设为同样的路径。

### 在Qt Creator中添加qt/embedded

工具->选项->设备，点击“添加”按钮，选择“通用Linux设备”，点击“开启向导”按钮，在弹出窗口中设置好名称、ip地址、ssh端口以及密码或密钥后，单击“下一步”并点击“完成”。

之后程序会主动进行ssh连接测试。

工具->选项->构建和运行->Qt版本，单击“添加”按钮，选择“/usr/local/Trolltech/QtEmbedded-4.8.6-arm/bin/qmake”，点击“应用”按钮。

切换到“构建套件(Kit)”选项卡，单击”添加”按钮，“设备类型”选择“通用Linux设备”，“设备”一栏选择刚才创建的设备，“编译器”一栏选择之前创建的“ARM GCC”，“Qt 版本”一栏选择刚才添加的Qt 4.8.6(QtEmbedded-4.8.6-arm)，点击“确定”按钮。

添加完成。


## 安装qvfb

qvfb需要用qt桌面版编译，此外还需要libxtst的支持。

首先安装libxtst：

```
 sudo apt-get install libxtst-dev
```

使用qtcreator打开qt-everywhere-opensource-src-4.8.6/tools/qvfb文件中的工程，在qvfb.pro文件中添加以下内容：

```
 INCLUDEPATH     += ../../include/QtCore
```

然后选择qt桌面版构建，将构建好的二进制文件复制进桌面版qt-4.8.6的bin目录下。

## 安装嵌入式X86_64平台的QT

```
./configure -embedded -qvfb -qt-gfx-qvfb -qt-kbd-qvfb -qt-mouse-qvfb -prefix /usr/local/Trolltech/QtEmbedded-4.8.6-i386
 make
 sudo make install
```

程序会安装在/usr/local/Trolltech/QtEmbedded-4.8.6-i386文件夹下。

### 为QtEmbedded-4.8.6-i386添加qwt-6.1.0

打开qwt源码包中的qwtconfig.pri文件,找到：
```
QWT_INSTALL_PREFIX    = /usr/local/qwt-$$QWT_VERSION
```
将其修改为：
```
QWT_INSTALL_PREFIX    = /usr/local/Trolltech/qwt-$$QWT_VERSION-Embedded-i386
```
找到：
```
QWT_CONFIG     += QwtOpenGL
QWT_CONFIG     += QwtDesigner
```
将其注释掉：
```
\#QWT_CONFIG     += QwtOpenGL
\#QWT_CONFIG     += QwtDesigner
```
逐条运行下列命令：

```
/usr/local/Trolltech/QtEmbedded-4.8.6-i386/bin/qmake
make
sudo make install
sudo cp /usr/local/Trolltech/qwt-6.1.0-Embedded-i386/lib/* /usr/local/Trolltech/QtEmbedded-4.8.6-i386/lib/
```

需要用到qwt时，在*.pro文件中添加以下两行：

```
LIBS += -lqwt
INCLUDEPATH += /usr/local/Trolltech/qwt-6.1.0-Embedded-i386/include
```
### 在Qt Creator中添加QtEmbedded-4.8.6-i386

工具->选项->构建和运行->Qt版本，单击“添加”按钮，选择“/usr/local/Trolltech/QtEmbedded-4.8.6-i386/bin/qmake”，点击“应用”按钮。

切换到“构建套件(Kit)”选项卡，单击”添加”按钮，“设备类型”选择“桌面”，“编译器”一栏选择“GCC (x86 64bit in /usr/bin)”，“Qt 版本”一栏选择刚才添加的Qt 4.8.6(QtEmbedded-4.8.6-i386)，点击“确定”按钮。

添加成功。

## 测试qvfb

### 启动：

<pre><code>
 qvfb -width 800 -height 480 -depth 16,24,32 -nocursor
</code></pre>

### 测试：

使用示例进行测试之前，需设置LD_LIBRARY_PATH变量：

```
 export LD_LIBRARY_PATH=/usr/local/Trolltech/QtEmbedded-4.8.6-i386/lib/
```

 如果不设置，运行时会报错：

```
undefined symbol: _ZN7QWidget8qwsEventEP8QWSEvent
```
原因在于此程序为嵌入式版的程序，运行时要连接嵌入式版本qt的库，但是不设置的话，此程序会连接桌面版的库（以后用QtEmbedded-4.8.6-i386套件构建时无需此步）。

```
 /usr/local/Trolltech/Qt-embedded-qvfb-4.8.6/examples/widgets/analogclock/analogclock -qws
```


## 为树莓派编译qt应用

运行时也要指定qt/embedded的库（将库拷贝到树莓派上），编译时指定连接路径。

例如，在.pro文件中添加如下内容：

```
LIBS += -Wl,-rpath,PATH/TO/QtEmbedded-4.8.6-arm/lib
```

## 为qt/embedded添加触摸屏支持

需要用到tslib：

下载：[tslib-1.0.tar.bz2
](http://sourceforge.net/projects/tslib.berlios/)

```
tar -xvf tslib-1.0.tar.bz2
```

编译安装之前需安装以下工具：
```
sudo apt-get install libtool
sudo apt-get install automake
sudo apt-get install autogen
sudo apt-get install autoconf
```
还需要对源码进行一些修改：

```
cd tslib-1.0
vim tests/ts_calibrate.c
```

在源文件 tslib-1.0/tests/ts_calibrate.c 中找到下列代码：

```
if ((calfile = getenv("TSLIB_CALIBFILE")) != NULL) {
     cal_fd = open (calfile, O_CREAT | O_RDWR);
} else {
   cal_fd = open ("/etc/pointercal", O_CREAT | O_RDWR);
}
```

修改为如下形式：

```
if ((calfile = getenv("TSLIB_CALIBFILE")) != NULL) {
    cal_fd = open (calfile, O_CREAT | O_RDWR, 0777);
} else {
    cal_fd = open ("/etc/pointercal", O_CREAT | O_RDWR, 0777);
}
```

编译安装：

```
./autogen.sh
echo "ac_cv_func_malloc_0_nonnull=yes" >arm-linux.cache
```

注意修改安装目录（将“/home/qianchen”替换为自己的目录）：

```
./configure --host=arm-linux-gnueabihf --cache-file=arm-linux.cache -prefix=/home/qianchen/tslib
make
make install
```

### 编译qt/embedded

下载：[qt-everywhere-opensource-src-4.8.6.tar.gz](http://download.qt-project.org/archive/qt/4.8/4.8.6/)

```
tar -xvzf qt-everywhere-opensource-src-4.8.6.tar.gz 
cd qt-everywhere-opensource-src-4.8.6
```

修改qmake.conf文件，选择之前安装的交叉编译编译工具链（即将相应工具替换为arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin下的工具:）：

```
vim mkspecs/qws/linux-armv6-g++/qmake.conf
%s/arm-linux/arm-linux-gnueabihf/g
```

添加以下内容（将“/home/qianchen”替换为自己的目录）：

```
QMAKE_CFLAGS += -I/home/qianchen/tslib/include
QMAKE_LFLAGS += -L/home/qianchen/tslib/lib -Wl,-rpath-link=/home/qianchen/tslib/lib
 ./configure -v -opensource -confirm-license -embedded arm  -xplatform qws/linux-armv6-g++ -no-xcursor -qt-mouse-tslib -I ~/tslib/include -L ~/tslib/lib
 make
```

数小时后……

```
 sudo make install
```

程序将会安装在/usr/local/Trolltech/QtEmbedded-4.8.6-arm文件夹下。之后将/usr/local/Trolltech/QtEmbedded-4.8.6-arm复制到目标板（树莓派）上，方便起见，设为同样的路径。

