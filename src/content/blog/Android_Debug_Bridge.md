---
pubDatetime: 2016-04-01
title: Android Debug Bridge(adb) 以及 WIFI调试
featured: false
draft: false
tags:
  - Android
description: Android Debug Bridge (ADB) 是一个非常有用的命令行工具，可以通过它与模拟器或者已连接的设备进行通信。
---

Android Debug Bridge (ADB) 是一个非常有用的命令行工具，可以通过它与模拟器或者已连接的设备进行通信。

<!-- more -->

本质上它是一个 **Client-Server** 程序，包含三个部分：

1. **Client**，客户端程序，运行在开发环境，可以通过任意的命令行工具执行adb命令来操作Client。
2. **Server**，服务端程序没，运行在开发环境的一个后台进程，它负责管理Client和模拟器或真机设备上的adb守护进程之间的通信。
3. **Daemon**，守护进程，运行在模拟器或Android真机设备的后台。

**ADB工具位于 {sdk}/platform-tools/**

打开一个adb Client之后，客户端程序会首先检查adb服务端进程是否正在运行。如果没有，先启动之。服务端程序启动之后会绑定到本地端口5037，监听Client发出的命令（所有的adb Client与Server的通信均通过5037端口进行）。然后 Server会与所有的模拟器或真机建立连接，之后就可以通过adb命令来控制所有的实例了。

**在使用adb之前，确保模拟器或真机已开启 “USB调试（USB debugging）”**

## 语法

```
adb [-d|-e|-s <serialNumber>] <command>
```

如果仅有一个模拟器或真机设备连接，adb 命令会默认发送到这个设备。如果有多个，就需要通过 “-d|-e|-s <serialNumber>”参数来限定命令的目标设备。

1. -d ：直接发送到通过USB连接的设备
2. -e ：直接发送到模拟器
3. -s [serialNumber] ：发送到指定序列号的设备

## 常用命令

### 查看连接设备

输出格式：[serialNumber] [state]

```
$ adb devices
List of devices attached
emulator-5554  device
953756558  device
```

### 安装app

如果有多台设备连接，需要参数 “-d|-e|-s <serialNumber>”

```
$ adb install <path_to_apk>
```

### 卸载app

```
adb unstall <package_name>
```

### 清除应用数据

```
adb shell pm clear <package_name>
```

### 导出文件

```
adb pull <remote> <local>
```

### 导入文件

```
adb push <local> <remote>
```

### 查看栈顶Activity

```
adb shell dumpsys activity top
```

### 查看所有安装应用的包名

```
adb shell pm list packages -f
```

### 查看Activity Manager State

```
adb shell dumpsys activity
```

### 关闭服务端

```
adb kill-server
```

## ADB的无线调试

### 通过WIFI连接设备进行调试：

1.  确保Android设备和开发电脑在同一个WIFI网络内
2.  通过USB将Android设备连接到电脑
3.  设置目标设备监听TCP/IP的5555端口

        $ adb tcpip 5555

4.  断开USB连接
5.  查询Android设备的IP地址
6.  根据IP地址连接目标设备

        $ adb connnet <device-ip-address>

7.  确认目标设备的连接情况

            $ adb device
            List of devices attached
            <device-ip-address>:5555  device

    连接完成。

### 如果adb连接被断开了：

1.  确保电脑和Android设备还处在同一WIFI网络内
2.  从 **步骤6** 开始进行重连，执行 adb connnet
3.  如果无效，关闭 adb server

           adb kill-server

    然后 **重新执行连接** 操作。

### 参考资料

[官方文档](https://developer.android.com/intl/zh-cn/tools/help/adb.html)
