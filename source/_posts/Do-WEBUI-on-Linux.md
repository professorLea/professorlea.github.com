---
title: Do WEBUI on Linux
date: 2016-08-10 16:17:40
tags: UITest
categories: 测试
---

#### 环境搭建
##### Xvfb介绍
> Xvfb is an X server that can run on machines with no display hardware and no physical input devices. It emulates a dumb framebuffer using virtual memory.
> https://www.x.org/archive/X11R7.6/doc/man/man1/Xvfb.1.xhtml

简单来说，就是模拟了window的图形化功能，所以如果想实例化FireFoxWebDriver，仍然需要安装一个firefox。

##### Xvfb安装
```powershell
root@classb-opsdevtest-in4:~# aptitude search xvfb
i   xvfb                                                               - Virtual Framebuffer 'fake' X server 
root@classb-opsdevtest-in4:~# aptitude install xvfb
```
##### Xvfb启动

```powershell
root@classb-opsdevtest-in4:~# Xvfb :1 -screen 1 1600x1200x16
root@classb-opsdevtest-in4:~# ps -ef | grep Xvfb
root     19118  4988  0 10:48 pts/6    00:00:00 grep Xvfb
root     24808     1  0 Jun28 pts/1    00:00:33 Xvfb :1 -screen 1 1600x1200x16 -nolisten tcp
```
命令的意思是：作为Server Number1 监听。有两个屏幕配置，默认是Screen0，width, height, and depth = 1280x1024x8，第二块屏幕配置为width, height, and depth = 1600x1200x16。

这里如果用自动启动脚本是更佳的：

```shell
root@classb-opsdevtest-in4:/etc/init.d# vim /etc/init.d/xvfb
#!/bin/bash
#chkconfig: 345 95 50
#description: Starts xvfb on display 1
if [ -z "$1" ]; then
    echo "`basename $0` {start|stop}"
    exit
fi
case "$1" in
    start)
    Xvfb :1 -screen 1 1600x1200x16 -nolisten tcp &
    export DISPLAY=:1
    echo 'export DISPLAY=:1' >> ~/.bashrc
    ;;
    stop)
    killall Xvfb
    ;;
esac
```

##### Firefox
###### 安装

```shell
root@classb-opsdevtest-in4:~# aptitude search firefox
i   firefox-esr                                                        - Mozilla Firefox web browser - Extended Support Release (ESR) 
root@classb-opsdevtest-in4:~# aptitude install firefox-esr
root@classb-opsdevtest-in4:~# firefox -v
Mozilla Firefox 45.2.0
```

###### 启动firefox

```shell
root@classb-opsdevtest-in4:~# firefox
Xlib:  extension "RANDR" missing on display ":1".
Xlib:  extension "RANDR" missing on display ":1".
```

到这里是不是有点懵比，界面呢？？ VNC server来解决这个问题

##### X11VNC server
###### 介绍
> x11vnc allows one to view remotely and interact with real X displays (i.e. a display corresponding to a physical monitor, keyboard, and mouse) with any VNC viewer.
> http://www.karlrunge.com/x11vnc/

###### 安装运行

```shell
root@classb-opsdevtest-in4:~# aptitude search X11VNC
i   x11vnc                                                             - VNC server to allow remote access to an existing X session  

root@classb-opsdevtest-in4:~# aptitude install x11vnc
root@classb-opsdevtest-in4:~# x11vnc -display :1 -xkb
```

参数解释，Display 1，就是上面Xvfb配置的1号屏幕。
-xkb，看到官网解释，是优化键盘输入
> http://www.karlrunge.com/x11vnc/x11vnc_opts.html

###### 安装VNC Viewer
这个就不多说了，http://www.realvnc.com/，来这里找到下载包安装即可。
通过VNC viewer链接 云主机，效果如下：
![Alt text](viewer.png)


#### 测试执行效果
##### 测试selenium

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import selenium
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.common.exceptions import NoSuchElementException, TimeoutException
browser = webdriver.Firefox()
browser.get("http://www.baidu.com")
t=browser.find_element_by_xpath("//div[contains(@id,'ftCon')]")
print t.text
```

执行效果如下：
![Alt text](CI.png)

成功

#### 问题
- 还未大范围执行，不知道稳定性如何。也不知道和window/Mac OX真实场景相比如何
- openid访问内网一般都需要将军令，这个需要解决。
- Chrome还未实验，不知能否走下去。

#### 展望
- 如果使用Docker，安装Ubuntu的镜像，应该可以快速解决上述环境搭建问题。