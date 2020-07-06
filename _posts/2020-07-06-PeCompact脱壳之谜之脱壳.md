---
layout:     post
title:      谜之脱壳法—PeCompact壳
subtitle:   PeCompact壳单步跑飞判定脱壳
date:       2020-07-06
author:     BY kexiaohei
header-img: img/pockr.jpg
catalog: true
tags:
    - frida
    - android hook
    - 工具
---
# 谜之脱壳法—PeCompact壳



## 1、查壳

![image-20200116150954040](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116150954040.png)

## 2、脱壳

```
脱壳方案：这个壳完美诠释了一点，就是脱壳时候程序肯定跑到004xxxx去，所以只要我们发现大跳跃或者运行到这里就算是脱壳成功了，然后一键OD脱壳就好
```

首先进入程序，不断F8往下走，发现程序跑飞了：

![image-20200116153413961](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116153413961.png)

那么就在这里下断点，重新开始程序，F9运行，F7一步进去。

![image-20200116153742232](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116153742232.png)

可以看到程序进入了这里，那么就继续往下F8执行：

![image-20200116154259972](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116154259972.png)

同样，下好断点后，执行到这里F7进去：

结果发现，程序不会重新运行，这里很迷，程序直接倒退到第二次断点的这里，这时候直接F7就进去了，后面就全部F8走一遍，若遇到这两个断点就进去

![image-20200116154935101](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116154935101.png)

发现这里就是004xxxx的样子，这样就找一下OEP，是不是存在大跳跃，首先按照以往经验就是找一下标点符号是"-"的，那么运行到这里

![image-20200116155243819](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116155243819.png)

![image-20200116155342704](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116155342704.png)

F7进去：

![image-20200116155436589](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116155436589.png)

使用Ctrl+A强行解析：

![image-20200116155500416](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116155500416.png)

这样就找到入口点了

然后OD一键脱壳就OJBK

![image-20200116155612608](PeCompact%E8%84%B1%E5%A3%B3%E4%B9%8B%E8%B0%9C%E4%B9%8B%E8%84%B1%E5%A3%B3.assets/image-20200116155612608.png)