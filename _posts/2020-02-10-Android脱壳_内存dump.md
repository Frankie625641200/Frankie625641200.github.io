---
layout:     post
title:      安卓逆向_手脱apk腾讯壳
subtitle:   安卓逆向破解——脱老版本腾讯壳之反调试内存dump大法
date:       2020-02-10
author:     BY kexiaohei
header-img: img/Android.jpg
catalog: true
tags:
    - 安卓逆向
    - 腾讯壳
    - 反调试
    - 内存dump
---
# 腾讯加固壳之动态手动脱壳系列

## 1、分析so壳

目标so文件：libshella-2.8.1.so

![image-20191220143621101](http://frankie.github.io/img/android-re/image-20191220143621101.png)

### 1.1、使用IDA打开时候若会提示：

![image-20191220143904586](http://frankie.github.io/img/android-re/image-20191220143904586.png)

![image-20191220144053750](http://frankie.github.io/img/android-re/image-20191220144053750.png)

这时候注意关键在section ignore

如果强行打开，则会出现混淆情况

因为elf文件可以没有section，所以我们就可以把section改为00

#### 1.1.1 强行打开

​		全程OK，强行读取，发现文件始终还是可以被反汇编

![image-20191220144451880](http://frankie.github.io/img/android-re/image-20191220144451880.png)

但是函数部分全部变成sub_xxx

#### 1.1.2 使用010editor修改

​		打开010editor，若未激活最好先激活

#### 1.1.3 使用ElF解析器，导入bt文件，然后启用

​		![image-20191220144854734](http://frankie.github.io/img/android-re/image-20191220144854734.png)

![image-20191220145022609](http://frankie.github.io/img/android-re/image-20191220145022609.png)

![image-20191220145129359](http://frankie.github.io/img/android-re/image-20191220145129359.png)

![image-20191220145539134](http://frankie.github.io/img/android-re/image-20191220145539134.png)

然后选中section_header_table

使用tool中的and操作改为hex的00

![image-20191220145756546](http://frankie.github.io/img/android-re/image-20191220145756546.png)

![image-20191220145847298](http://frankie.github.io/img/android-re/image-20191220145847298.png)

#### 1.1.4 另存为并重新放入ida

![image-20191220150108557](http://frankie.github.io/img/android-re/image-20191220150108557.png)

已经正常显示，这时候就可以静态分析了

### 1.2、jni函数被加密

![image-20200107141347169](http://frankie.github.io/img/android-re/image-20200107141347169.png)

​		由于我们知道so文件加载的时候会首先查看.init或.init_array段是否存在，如果存在那么先运行这段内容，如果不存在那么就检查是否存在JNI_onload()，如果存在则执行jni_onload()。

​		所以我们可以推测，解密jni_onload()应该就在.init或.init_array处。

​		因为动态调试so的时候，jni_onload()是可以显示，但是.init或.init_array是无法显示的，所以必须通过静态分析出这两段偏移量然后动态调试的时候计算出绝对位置，然后再make code（快捷键：c），这样才可以看到该段内代码内容

#### 1.2.1 获取so文件中的.init或.init_array

其中两个方法我未尝试成功，不过可以通过linux读取：

```
readelf libshella.so -a
```

![image-20200107144049734](http://frankie.github.io/img/android-re/image-20200107144049734.png)

#### 1.2.2 分析.init_array

​		找到.init函数执行位置，使用IDA查看：

![image-20200107144603834](http://frankie.github.io/img/android-re/image-20200107144603834.png)

（使用快捷键g）直接搜索

接下来进入到sub_14D4

![image-20200107144701045](http://frankie.github.io/img/android-re/image-20200107144701045.png)

F5编译一下：

![image-20200107145013714](http://frankie.github.io/img/android-re/image-20200107145013714.png)



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



![image-20200107233020028](http://frankie.github.io/img/android-re/image-20200107233020028.png)

这时候将基址加上偏移地址

即base的值加上2748（待考究是否固定）（考究完毕：查看文章偏移量计算）

4003C000 + 2748 =4003E748

![image-20200108001024604](http://frankie.github.io/img/android-re/image-20200108001024604.png)

找到位置即可用快捷键C

## 3、下断点

![image-20200108001236927](http://frankie.github.io/img/android-re/image-20200108001236927.png)

启动：

![image-20200108001420687](http://frankie.github.io/img/android-re/image-20200108001420687.png)

运行后在节点处停下进行下一步分析

## 4、流程式查看

```
F9到运行在BL        unk_400405F4            使用F7进去

看到第一个初始化函数；
```

**这里需要注意，可以看到前面是libshella.so而不是其它的so文件，其中几次调试会出现libBugly.so的so文件**

![image-20200219235418284](http://frankie.github.io/img/android-re/image-20200219235418284.png)

进入libshella.so前：

![image-20200108004921178](http://frankie.github.io/img/android-re/image-20200108004921178.png)

```
由于没有逻辑处理，紧接着F8回来，再一次F9再一次F7进去：
```

这里点击P键恢复原函数

```
可以看到第二个初始化函数：
```

![image-20200108005045687](http://frankie.github.io/img/android-re/image-20200108005045687.png)

同步SP寄存器查看就可以看到程序执行流程了

在这里可以跟随函数不断往下走，也可以试试脚本

```c
static main(void)
{
    do
    {
        step_over();
        wait_for_next_event(STEP,-1);
    }while(PC!=0xEEDAFC2C);//停在何处
}
```

单步跟踪法查询到查询执行死亡的位置，然后定位

![image-20200220002835937](http://frankie.github.io/img/android-re/image-20200220002835937.png)

进入后，点击P分析代码

![image-20200220003204485](http://frankie.github.io/img/android-re/image-20200220003204485.png)

![image-20200220003240063](http://frankie.github.io/img/android-re/image-20200220003240063.png)

## 5、nop掉反调试

定位寻找函数create()相关的创建线程的反调试函数；

而定位后需要对其进行nop，即将16进制的hex数据改为**00 00 A0 E1**

每一个create函数都会被隐写，因此都需要c键去分析一下，如下unk_7822835C

进去后，按下C键，从而确定为create函数，然后esc键返回。

![image-20200220003950687](http://frankie.github.io/img/android-re/image-20200220003950687.png)

![image-20200220005235476](http://frankie.github.io/img/android-re/image-20200220005235476.png)

![image-20200220005308825](http://frankie.github.io/img/android-re/image-20200220005308825.png)

然后定位create改为nop函数

![image-20200220005418907](http://frankie.github.io/img/android-re/image-20200220005418907.png)

![image-20200220005450985](http://frankie.github.io/img/android-re/image-20200220005450985.png)

然后根据libdvm.so计算偏移量：

414CE000+手机本机偏移量计算得偏移量得值=4151DFAC（查阅文章arm手机偏移量）

![image-20200220005538266](http://frankie.github.io/img/android-re/image-20200220005538266.png)

定位jni_onload()的调用文件点：

![image-20200220005816957](http://frankie.github.io/img/android-re/image-20200220005816957.png)

运行到此处后F7进入：

![image-20200220005954048](http://frankie.github.io/img/android-re/image-20200220005954048.png)

P一下后F5查看伪代码查询逻辑，发现调用dlopen打开动态链接库和dlsym返回符号地址

因此这个函数不仅仅是获取函数地址，还可以获取变量地址

在反汇编窗口，运行完程序后找到一个关键跳转点：

在这里return返回前进行了一个逻辑处理：

![image-20200221000759111](http://frankie.github.io/img/android-re/image-20200221000759111.png)

选择F7跟进处理查看，这里可见需要关注的大多为寄存器的存储值，除此之外，还需要每个函数跟进分析：

![image-20200221001025725](http://frankie.github.io/img/android-re/image-20200221001025725.png)

在判断以后执行最后一个:

![image-20200221001224040](http://frankie.github.io/img/android-re/image-20200221001224040.png)

而进去后为：

找到了Android的大多数信息：

![image-20200221001304139](http://frankie.github.io/img/android-re/image-20200221001304139.png)

却不是执行程序，然后就看第二个函数：

![image-20200221001352827](http://frankie.github.io/img/android-re/image-20200221001352827.png)

第二个函数传入v4，而v4为：

![image-20200221001501520](http://frankie.github.io/img/android-re/image-20200221001501520.png)

![image-20200221001417200](http://frankie.github.io/img/android-re/image-20200221001417200.png)

load done！为何load？肯定是用了一些基址函数去读取，继续跟随

![image-20200221001604193](http://frankie.github.io/img/android-re/image-20200221001604193.png)

可见，v4相当于a1传入，如果是成功的，执行了第五行程序，而第五行程序为什么程序呢？

跟踪查看：

![image-20200221001809338](http://frankie.github.io/img/android-re/image-20200221001809338.png)

可见，传入的4个值均为注册，而v1程序里，未知的那个值就是读取的dex文件

因此可以推理出，如果这个时候secshell加载成功，dex文件就会被隐藏

我们dump点只能在执行过程后的那一刻，而必须是在执行过程，因此，对v1程序里的那个未知参数进行跟踪：

![image-20200221002029382](http://frankie.github.io/img/android-re/image-20200221002029382.png)

点D键分析或直接跟踪：

![image-20200221002137173](http://frankie.github.io/img/android-re/image-20200221002137173.png)

## 6、找到关键的线程函数

发现是关键的runCreate，而对应的参数是78017D96

因此我们就要点击为上一个

![image-20200221002517275](http://frankie.github.io/img/android-re/image-20200221002517275.png)



![image-20200221002614680](http://frankie.github.io/img/android-re/image-20200221002614680.png)

进入到这里secshell的读取位置，这时候如果有经验的话，直接可以看到关键点在result，而result最后赋值hi在判断v8以后的

而相对较好就是每一个程序都跟踪一次

![image-20200221002651270](http://frankie.github.io/img/android-re/image-20200221002651270.png)

跟踪v8为判断java的vm类型，是否存在libart.so文件，这时候如果模块中没有libart.so文件就说明执行的是23行这个分支：

![image-20200221002910777](http://frankie.github.io/img/android-re/image-20200221002910777.png)

因此，在23行跳进去函数后直接在其实头压栈的时候下好断点：

![image-20200221003100333](http://frankie.github.io/img/android-re/image-20200221003100333.png)

## 7、查看分析关键函数

然后查看程序：

![image-20200221003748345](http://frankie.github.io/img/android-re/image-20200221003748345.png)

得出：

![image-20200221004238320](http://frankie.github.io/img/android-re/image-20200221004238320.png)

这个时候进行流程图查看：

![image-20200221004721959](http://frankie.github.io/img/android-re/image-20200221004721959.png)

通过执行程序流程图，跟踪程序执行流程：

![image-20200221004904355](http://frankie.github.io/img/android-re/image-20200221004904355.png)

看到这里开始关注寄存器，堆栈，还有汇编的变化，记录每一步的变化：

![image-20200221005032910](http://frankie.github.io/img/android-re/image-20200221005032910.png)

关注点，跟踪解密特殊位置函数，已知数据大小为0x28即40：

![image-20200221005215166](http://frankie.github.io/img/android-re/image-20200221005215166.png)

跟踪计算偏移量，进行解密

![image-20200221005456160](http://frankie.github.io/img/android-re/image-20200221005456160.png)

**可知计算下来的执行下来的R3=0x38C14，R2=0x9710，然后R0=R3+R2 ，然后R3=0x1000 , 然后R0=R0+R3=(0x38C14+0x9710)+0x1000 ，然后逻辑右移0xC ， 再逻辑左移0xC**

因此写出python程序：

```python
hex((((0x38C14+0x9710)+0x1000)>>0xC)<<0xC)
```

得到结果：

![image-20200221010151784](http://frankie.github.io/img/android-re/image-20200221010151784.png)

为0x43000

而数据头文件大小为0x43000+0x28=0x43028

跟踪R0

![image-20200221010712214](http://frankie.github.io/img/android-re/image-20200221010712214.png)

这时候为了方便计算

![image-20200221010818958](http://frankie.github.io/img/android-re/image-20200221010818958.png)

先dump下dex

![image-20200221011017453](http://frankie.github.io/img/android-re/image-20200221011017453.png)

```C
static main(void)
{
  auto fp, begin, end, dexbyte;
  fp = fopen("C:\\dump.dex", "wb");
  begin = 0x774CE000;
  end = 0x775C2000;
  for ( dexbyte = begin; dexbyte < end; dexbyte ++ )
      fputc(Byte(dexbyte), fp);
}
```

## 8、处理dex文件

可以看到，混淆到我都不知道是odex还是dex文件了

![image-20200221011220177](http://frankie.github.io/img/android-re/image-20200221011220177.png)

然后继续看程序，执行解密heard部分数据：

![image-20200221012218089](http://frankie.github.io/img/android-re/image-20200221012218089.png)

这里对R5进行处理：

![image-20200221012333581](http://frankie.github.io/img/android-re/image-20200221012333581.png)

双击R5跟踪：

![image-20200221012423091](http://frankie.github.io/img/android-re/image-20200221012423091.png)

这里竟然出现了dex

说明这里就是起始点，那么还有就是结束点

结束点=起始点+文件大小，应该就在源程序中：

跟踪每一个可疑的数据：

![image-20200221012717764](http://frankie.github.io/img/android-re/image-20200221012717764.png)

看到R2是0x70就是文件大小，因为：程序执行四个参数分别为寄存器的r0,r1,r2,r3

![image-20200221012813482](http://frankie.github.io/img/android-re/image-20200221012813482.png)

而在输出的时候显示：

![image-20200221012914000](http://frankie.github.io/img/android-re/image-20200221012914000.png)

而这个时候还有一个问题就是第一个程序的文件大小，这样只有偏移量没有大小

这时候又得回到源程序

![image-20200221013104810](http://frankie.github.io/img/android-re/image-20200221013104810.png)

可以看到这里面引入了一个新的变量v70=v00,然后v70执行了一系列操作

不管那么多，先执行，发现v100指向R3

R3为A7C14

![image-20200221013338267](http://frankie.github.io/img/android-re/image-20200221013338267.png)

若要分析v100是否为文件大小，向上逻辑虽然找不到，但是至少v70 跟踪可以看出

先dump下dex heard文件

![image-20200221014146144](http://frankie.github.io/img/android-re/image-20200221014146144.png)

脚本：

```C
static main(void)
{
  auto fp, begin, end, dexbyte;
  fp = fopen("C:\\dump_heard.dex", "wb");
  begin = 0xBED8EC54;
  end = 0xBED8EC54+0x70;
  for ( dexbyte = begin; dexbyte < end; dexbyte ++ )
      fputc(Byte(dexbyte), fp);
}
```

因此程序逻辑清晰了：

**先获取到混淆的dex，然后根据偏移量，计算出起始点位置，然后再解密另一个dex文件头部和文件大小，然后拼接处再前面的dex的偏移量起始点处，这时候在赋上第一个文件大小A7C14**

然后使用010editor：

edit->select range

![image-20200221014746694](http://frankie.github.io/img/android-re/image-20200221014746694.png)

然后File->save selection as file 

![image-20200221015225889](http://frankie.github.io/img/android-re/image-20200221015225889.png)

保存后，到dexheard.dex中全选，edit ->copy as hex

然后到前面保存好的dex中paste from hex

![image-20200221015546172](http://frankie.github.io/img/android-re/image-20200221015546172.png)



打开dex

![image-20200221015619232](http://frankie.github.io/img/android-re/image-20200221015619232.png)

完美脱壳

