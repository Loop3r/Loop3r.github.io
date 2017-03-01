---
layout: post
title: "OpenWrt下设置程序开机自启"
date: 2016-05-08
tags: OpenWrt
---

## 在/etc/init.d/中按照以下格式编写shell脚本

    #!/bin/sh /etc/rc.common
    # Example script
    # Copyright (C) 2007 OpenWrt.org
     
    START=10
    STOP=15
     
    start() {        
            echo start
            # commands to launch application
    }                 
     
    stop() {          
            echo stop
            # commands to kill application 
    }

以上便是一份在自启动的shell脚本模板。在start方法中写入运行程序的命令，而stop方法中写入终止程序运行的命令即可。在上面的代码中，第一行称为shebang line，它使用/etc/rc.common脚本作为包装器。第二行和第三行的START=99, STOP=15指的是优先级，优先级的脚本会先运行。数字越大，优先级越低。

## 使能脚本
1. 使用chmod命令将脚本变为可执行脚本：*chmod +x xxx*
2. 使用*xxx enable*使得脚本开机自启动。该命令实质上是为脚本文件创建一个软链接，软链接存放于/etc/rc.d/下，如果我们不想使用rc.common的enable命令也可以，我们可以自己创建链接。
3. 通过以上的步骤就可以创建程序自启动脚本，将程序设置为自启动。

3. 另外，如果在开机boot期间，需要运行程序，我们可以使用boot方法。使用方法类似于start方法和stop方法。


[参考openwrt官方文档](http://wiki.openwrt.org/doc/techref/initscripts)