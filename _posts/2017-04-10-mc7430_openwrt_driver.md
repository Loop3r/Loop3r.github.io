---
layout: post
title: "sierra wireless mc7430 4g模块openwrt平台驱动流程"
date: 2017-04-10 
tags: Openwrt  
---
#### 将驱动build in openwrt固件
1. 使用官方提供的驱动GobiNet和GobiSerial，不要使用openwrt自带的驱动。
2. `make menuconfig`进入openwrt固件配置菜单，选中以下依赖包：
```
Kernel Modules -> USB Support
    <*> kmod-usb2
    <*> kmod-usb-ohci
    <*> kmod-usb-uhci
    <*> kmod-usb-acm # For ACM based modem, such as Nokia Phones
    <*> kmod-usb-net # For tethering and rndis support
    <*> kmod-usb-serial..................... Support for USB-to-Serial converters 
```
3. 参照rt5350驱动开发教程，在`/trunk/package/kernel`目录下建立如下文件夹：
```
/trunk/
    package/
        kernel/
            GobiNet/
                src/
                makefile
            GobiSerial/
                src/
                makefile
```
将`GobiNet`和`GobiSerial`的源文件分别添加到相应的`src/`文件夹。
4. makefile内容分别如下：

---

GobiNet/makefile
```
#
# Copyright (C) 2008-2012 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=GobiNet
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define KernelPackage/GobiNet
  SUBMENU:=Other modules
  DEPENDS:=+kmod-usb-core +kmod-usb-net
  TITLE:=Sierra mc7430 GobiNet driver
  FILES:=$(PKG_BUILD_DIR)/GobiNet.ko
  AUTOLOAD:=$(call AutoLoad,30,GobiNet,1)
  KCONFIG:=
endef

define KernelPackage/GobiNet/description
 This is a GobiNet drivers
 endef

MAKE_OPTS:= \
	ARCH="$(LINUX_KARCH)" \
	CROSS_COMPILE="$(TARGET_CROSS)" \
	SUBDIRS="$(PKG_BUILD_DIR)" \
	RAWIP=1

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Compile
	$(MAKE) -C "$(LINUX_DIR)" \
		$(MAKE_OPTS) \
		modules
endef

$(eval $(call KernelPackage,GobiNet))
```
---

GobiSerial/makefile
```
#
# Copyright (C) 2008-2012 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=GobiSerial
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define KernelPackage/GobiSerial
  SUBMENU:=Other modules
  DEPENDS:=+kmod-usb-core +kmod-usb-net +kmod-usb-serial
  TITLE:=Sierra mc7430 GobiSerial driver
  FILES:=$(PKG_BUILD_DIR)/GobiSerial.ko
  AUTOLOAD:=$(call AutoLoad,30,GobiSerial,1)
  KCONFIG:=
endef

define KernelPackage/GobiSerial/description
 This is a GobiSerial drivers
 endef

MAKE_OPTS:= \
	ARCH="$(LINUX_KARCH)" \
	CROSS_COMPILE="$(TARGET_CROSS)" \
	SUBDIRS="$(PKG_BUILD_DIR)"

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Compile
	$(MAKE) -C "$(LINUX_DIR)" \
		$(MAKE_OPTS) \
		modules
endef

$(eval $(call KernelPackage,GobiSerial))
```
5. `make menuconfig`进入配置菜单选中
```
Kernel Modules -> Other modules
    <*> kmod-GobiNet
    <*> kmod-GobiSerial
```
然后编译固件并烧写至开发板。
6. 将4g模块连接开发板并开机上电，如开机log中出现
```
[   10.020000] usb 1-1: no of_node; not parsing pinctrl DT
[   10.030000] GobiNet 1-1:1.12: no of_node; not parsing pinctrl DT
[   10.030000] GobiSerial 1-1:1.12: no of_node; not parsing pinctrl DT
[   10.030000] GobiNet 1-1:1.13: no of_node; not parsing pinctrl DT
[   10.030000] GobiSerial 1-1:1.13: no of_node; not parsing pinctrl DT
```
需要使用`SetUsbComp.exe`软件修改模块的参数方能正常驱动。

#### PC连接开发板的4G模块上网
1. PC通过网线连接开发板LAN口，并设置为自动获取IP地址。
2. `microcom -s /dev/ttyUSB2`
3. 使用`at!gstatus?`指令查看`PS state`是否为`attached`。
4. `at!scact?`
5. `at!scact=1,1`
6. `vi /etc/config/network`修改网络配置，将`wan`、`wan6`接口的`ifname`改为`eth1`。
7. `udhcpc -i eth1`