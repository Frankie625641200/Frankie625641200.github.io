---
layout:     post
title:      安卓逆向_脱apk腾讯壳
subtitle:   安卓逆向破解——脱老版本腾讯壳之反调试内存dump大法
date:       2020-02-10
author:     BY kexiaohei
header-img: img/Android.jpg
catalog: true
tags:
    - 安卓逆向
    - 腾讯壳
    - 反调试
---

# 腾讯加固壳之动态脱壳系列

## 1、分析so壳

目标so文件：libshella-2.8.1.so

![image-20191220143621101](http:frankie625641200.github.io/img/android-re/image-20191220143621101.png)

### 1.1、使用IDA打开时候若会提示：

![image-20191220143904586](http:frankie625641200.github.io/img/android-re/image-20191220143904586.png)

![image-20191220144053750](http:frankie625641200.github.io/img/android-re/image-20191220144053750.png)

这时候注意关键在section ignore

如果强行打开，则会出现混淆情况

因为elf文件可以没有section，所以我们就可以把section改为00

#### 1.1.1 强行打开

​		全程OK，强行读取，发现文件始终还是可以被反汇编

![image-20191220144451880](http:frankie625641200.github.io/img/android-re/image-20191220144451880.png)

但是函数部分全部变成sub_xxx

#### 1.1.2 使用010editor修改

​		打开010editor，若未激活最好先激活

#### 1.1.3 使用ElF解析器，导入bt文件，然后启用

​		![image-20191220144854734](http:frankie625641200.github.io/img/android-re/image-20191220144854734.png)

![image-20191220145022609](http:frankie625641200.github.io/img/android-re/image-20191220145022609.png)

![image-20191220145129359](http:frankie625641200.github.io/img/android-re/image-20191220145129359.png)

![image-20191220145539134](http:frankie625641200.github.io/img/android-re/image-20191220145539134.png)

然后选中section_header_table

使用tool中的and操作改为hex的00

![image-20191220145756546](http:frankie625641200.github.io/img/android-re/image-20191220145756546.png)

![image-20191220145847298](http:frankie625641200.github.io/img/android-re/image-20191220145847298.png)

#### 1.1.4 另存为并重新放入ida

![image-20191220150108557](http:frankie625641200.github.io/img/android-re/image-20191220150108557.png)

已经正常显示，这时候就可以静态分析了

### 1.2、jni函数被加密

![image-20200107141347169](http:frankie625641200.github.io/img/android-re/image-20200107141347169.png)

​		由于我们知道so文件加载的时候会首先查看.init或.init_array段是否存在，如果存在那么先运行这段内容，如果不存在那么就检查是否存在JNI_onload()，如果存在则执行jni_onload()。

​		所以我们可以推测，解密jni_onload()应该就在.init或.init_array处。

​		因为动态调试so的时候，jni_onload()是可以显示，但是.init或.init_array是无法显示的，所以必须通过静态分析出这两段偏移量然后动态调试的时候计算出绝对位置，然后再make code（快捷键：c），这样才可以看到该段内代码内容

#### 1.2.1 获取so文件中的.init或.init_array

其中两个方法我未尝试成功，不过可以通过linux读取：

```
readelf libshella.so -a
```

![image-20200107144049734](http:frankie625641200.github.io/img/android-re/image-20200107144049734.png)

#### 1.2.2 分析.init_array

​		找到.init函数执行位置，使用IDA查看：

![image-20200107144603834](http:frankie625641200.github.io/img/android-re/image-20200107144603834.png)

（使用快捷键g）直接搜索

接下来进入到sub_14D4

![image-20200107144701045](http:frankie625641200.github.io/img/android-re/image-20200107144701045.png)

F5编译一下：

![image-20200107145013714](http:frankie625641200.github.io/img/android-re/image-20200107145013714.png)



## 2、启动IDA动态调试

```
adb shell 
su 
/data/local/tmp/as -p31928

adb forward tcp:31928 tcp:31928

adb shell am start -D -n com.qianyu.app/.LoginActivity

adb forward tcp:8700 jdwp:15127

jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8652

如果出现错误
查看serverSocket所监听的端口
netstat -nao

关键地址:基址+偏移地址
```



![image-20200107233020028](http:frankie625641200.github.io/img/android-re/image-20200107233020028.png)

这时候将基址加上偏移地址

即base的值加上2748（详细数据获取：查看文章偏移量计算）

4003C000 + 2748 =4003E748

![image-20200108001024604](http:frankie625641200.github.io/img/android-re/image-20200108001024604.png)

找到位置即可用快捷键C

## 3、下断点

![image-20200108001236927](http:frankie625641200.github.io/img/android-re/image-20200108001236927.png)

启动：

![image-20200108001420687](http:frankie625641200.github.io/img/android-re/image-20200108001420687.png)

运行后在节点处停下进行下一步分析

## 4、流程式查看

```
F9到运行在BL        unk_400405F4            使用F7进去

看到第一个初始化函数；
```

![image-20200108004921178](http:frankie625641200.github.io/img/android-re/image-20200108004921178.png)

```
由于没有逻辑处理，紧接着F8回来，再一次F9再一次F7进去：
```



```
可以看到第二个初始化函数：
```

![image-20200108005045687](http:frankie625641200.github.io/img/android-re/image-20200108005045687.png)

同步SP寄存器查看就可以看到程序执行流程了

利用搜索（Alt+T）

搜索jni_onload，快捷键c一下：

![image-20200108005447105](http:frankie625641200.github.io/img/android-re/image-20200108005447105.png)

下好断点

![image-20200109004056964](http:frankie625641200.github.io/img/android-re/image-20200109004056964.png)

执行进去后

![image-20200109004148366](http:frankie625641200.github.io/img/android-re/image-20200109004148366.png)

然后直接F5编译查看，然后copy过去：

![image-20200109004549808](http:frankie625641200.github.io/img/android-re/image-20200109004549808.png)





![image-20200113123437698](http:frankie625641200.github.io/img/android-re/image-20200113123437698.png)

R8进入后找到R2寄存器进入点，F7进入分析