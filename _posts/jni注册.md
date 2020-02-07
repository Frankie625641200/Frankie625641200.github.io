---
layout:     post
title:      jni注册
subtitle:   Android编写C++使用JNI注册
date:       2020-02-8
author:     BY kexiaohei
header-img: img/Android.jpg
catalog: true
tags:
    - Android
    - jni
    - Java&C++
    - so文件
---
# 前言

>在Android开发过程中经常遇到Java与C++/C交互的做法，其过程是生成了SO文件，中间就是使用了jni方法

# 环境
操作系统：windows10

操作环境：JAVA、NDK

工具：Android studio

# 操作
## JNI注册

### 1、jni注册需要一开始java中有函数

一开始在程序中调用其中的函数：

```java
public native String helloworld();
```

![image-20200201202623252](img/jni/image-20200201202623252.png)

### 2、使用javah命令去执行生成C语言头文件.h

```
javah -jni -encoding UTF-8 com.finalexample.test2.MainActivity
```

#### 	2.1 需要注意的是这里的java必须使用JDK10以下版本，最好是JDK8

然后注意是跳进项目后进入com开始的包名开始进行生成：

![image-20200201203104316](img/jni/image-20200201203104316.png)

### 3、创建JNI文件夹，并移动头文件

![image-20200201203507193](img/jni/image-20200201203507193.png)

### 4、写入两个文件Android.mk和Application.mk入JNI文件夹

![image-20200201203717981](img/jni/image-20200201203717981.png)

Android.mk：

```makefile
LOCAL_PATH := $(call my-dir)  
include $(CLEAR_VARS) 
local_arm_mode:= arm #设置为ARM指令 
LOCAL_MODULE    := helloworld  #模块名称                     需要更改
LOCAL_SRC_FILES := helloworld.c #源文件.c或者.cpp                 需要更改
LOCAL_LDLIBS += -llog   #依赖库  
include $(BUILD_SHARED_LIBRARY) #指定编译文件类型  这里是动态链接库 .so文件
```

Application.mk:

```makefile
APP_ABI := armeabi-v7a
```

### 5、重命名

![image-20200201204500604](img/jni/image-20200201204500604.png)

### 6、新建一个c文件并包含头文件

```
#include "aescoder.h"
```

![image-20200201205720447](img/jni/image-20200201205720447.png)

### 7、写入第一个程序

```
JNIEXPORT jstring JNICALL Java_com_finalexample_test2_MainActivity_aescoder
(JNIEnv * env, jobject obj){

  return (*env)->NewStringUTF(env,"aaaa");
  }
```

### 8、进入到jni目录下使用ndk-build

![image-20200201205913817](img/jni/image-20200201205913817.png)

### 9、然后就可以生成so文件

![image-20200201210023490](img/jni/image-20200201210023490.png)

### 10、调用

![image-20200201210410596](img/jni/image-20200201210410596.png)

## Hey