---
layout:     post
title:      frida环境+脚本
subtitle:   frida搭建与使用
date:       2020-07-05
author:     BY kexiaohei
header-img: img/Android.jpg
catalog: true
tags:
    - frida
    - android hook
    - 工具
---
# ANACONDA+frida

## 1、环境下载

下载版本：

windows(x64)+python3.7        

frida版本通过anaconda环境中的python3.7中安装的版本进行对应下载

```
anaconda下载地址：https://www.anaconda.com/distribution/
frida-server下载地址：https://github.com/frida/frida/releases
```

frida版本确认：

**安装frida过程最好翻墙下载**

```python
pip install frida
pip install frida-tools
frida --v
```

![image-20200504192033582](http:frankie625641200.github.io/img/image-20200504192033582.png)

下载对应的frida-server

## 2、环境搭建

```
D:\ProgramData\Anaconda3;
D:\ProgramData\Anaconda3\Library\bin;
D:\ProgramData\Anaconda3\Library\mingw-w64\bin;
D:\ProgramData\Anaconda3\Scripts;
```

### 2.1  启动frida server 转发端口

​			改包名与命令一致

​			端口转发：adb forward tcp:27042 tcp:27042

### 2.2 启动app

​			通过frida去启动app测试

包名：com.xiaojianbang.app

```
frida -U -f com.xiaojianbang.app --no-pause（启动app）
```

![image-20200504193405269](http:frankie625641200.github.io/img/image-20200504193405269.png)





# 2、脚本运行

使用spyder编辑相关脚本

```python
# -*- coding: utf-8 -*-
"""
Spyder Editor

This is a temporary script file.

JAVA层HOOK
"""

import frida, sys

#HOOK普通方法

jscode = """
Java.perform(function () {
    var utils = Java.use('com.xiaojianbang.app.Utils');
    utils.getCalc.implementation = function (a, b) {
        console.log("Hook Start...");
		send(arguments[0]);
        send(arguments[1]);
        send("Success!");
		var num = this._getCalc(1000, 2000, 3000);
		send(num);
		return num;
    }
});
"""

#HOOK构造方法
jscode = """
Java.perform(function () {
	var money = Java.use('com.xiaojianbang.app.Money');
    money.$init.implementation = function (a, b) {
        console.log("Hook Start...");
		send(arguments[0]);
		send(arguments[1]);
        send("Success!");
		return this.$init(10000, "美元");
    }
});
"""

#HOOK重载方法
jscode = """
Java.perform(function () {
    var utils = Java.use('com.xiaojianbang.app.Utils');
    utils.test.overload("int").implementation = function (a) {
        console.log("Hook Start...");
		send(arguments[0]);
        send("Success!");
		return "qianyu";
    }
});
"""

def message(message, data):
    if message["type"] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)

process = frida.get_remote_device().attach('com.xiaojianbang.app')
script= process.create_script(jscode)
script.on("message", message)
script.load()
sys.stdin.read()

```

frida.get_remote_device().attach("")添加包名

console.log("Hook Start...");输出

var utils = Java.use()类的位置：包名+类名

 utils.getCalc.implementation = function (a, b) 添加参数个数

.overload("int")特殊情况加入变量改变











# 附录

```python
# -*- coding: utf-8 -*-
"""
Spyder 编辑器

这是一个临时脚本文件。
"""

import frida
import sys


jscode="""
Java.perform(function(){
    var until = Java.use('com.xiaojianbang.app.Utils');
    until.test.overload("int").implementation = function(a){
        console.log("start");
        send(arguments[0])
        send(this.test(arguments[0]));
        return "hhhhh";
    }

}
);


"""



def message(message, data):
    if message["type"] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)

process = frida.get_remote_device().attach('com.xiaojianbang.app')
script= process.create_script(jscode)
script.on("message", message)
script.load()
sys.stdin.read()

```

