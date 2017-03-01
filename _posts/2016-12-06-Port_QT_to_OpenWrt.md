---
layout: post
title: "移植QT到OpenWrt"
date: 2016-12-06 
tags: OpenWrt  
---

#### 编译tslib
修改Makefile：CC=.../mipsel-linux-gcc；CXX=.../mipsel-linux-g++
    
    ./autogen.sh
    ./configure --prefix=/home/champer/tslib/  --host=mips-linux  ac_cv_func_malloc_0_nonnull=yes
    make & make install
make时出现错误：

    error: call to ‘__open_missing_mode’ declared with attribute error: open with O_CREAT in second argument needs 3 arguments
错误原因及解决办法：gcc语法检查严格，必须加上第三个参数 --0777

#### 编译qt
从qt官网下载qt-everywhere-opensource-src-4.8.1.tar.gz并解压。

修改/mkspecs/qws/linux-mips-g++/qmake.conf如下

    # modifications to g++.conf
    QMAKE_CC                = mipsel-openwrt-linux-gcc
    QMAKE_CXX               = mipsel-openwrt-linux-g++
    QMAKE_CFLAGS           += -mips32
    QMAKE_CXXFLAGS         += -mips32
    QMAKE_LINK              = mipsel-openwrt-linux-g++
    QMAKE_LINK_SHLIB        = mipsel-openwrt-linux-g++
    # modifications to linux.conf
    QMAKE_AR                = mipsel-openwrt-linux-ar cqs
    QMAKE_OBJCOPY           = mipsel-openwrt-linux-objcopy
    QMAKE_STRIP             = mipsel-openwrt-linux-strip
然后运行configure

    ./configure -embedded mips -xplatform qws/linux-mips-g++ -prefix /home/champer/qt-mips -release -opensource -confirm-license -webkit -no-qt3support -make libs -nomake examples -nomake demos -little-endian

最后make & make install，其间出错：error: invalid conversion from 'int*' to 'socklen_t*'。找到定义QT_SOCKLEN_T的文件/mkspecs/linux-g++/qplatformdefs.h,作如下修改：

    #define QT_SOCKLEN_T            socklen_t

现在就可以使用/home/champer/tslib/bin/qmake进行编译了。将编译好的程序与相关库文件传入开发板即可运行。
