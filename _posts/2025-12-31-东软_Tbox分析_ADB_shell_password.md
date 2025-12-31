---
layout:     post
title:      东软_Tbox分析_ADB_shell_password
subtitle:  通过对tbox零部件进行测试，发现切入adb可以被测试
date:       2023-03-03
author:     BY kexiaohei
header-img: img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/%E5%9B%BE%E7%89%871.png
catalog: true
tags:
    - Tbox
    - 东软
    - router
    - 命令执行
---
# 东软\_Tbox分析\_ADB\_shell\_password





## 0x01 启动

​		首先是拿到引脚定义，看到电源，定位接12V，这里直接可以接出来，大部分都是斜对角

![image-20231101135930267](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101135930267.png)

这时候通电给Tbox就可以启动起来了

这里可以到还有个调试口，其实不难发现，这里有个通用得转接口，直接转USB使用adb调试

![图片1](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/%E5%9B%BE%E7%89%871.png)

![image-20231101141502535](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101141502535.png)

整理是需要输入mdm9607 login



## 0x02 提取固件

​		这里有个bug，发现其实不给执行adb shell情况下，是可以实现adb push和adb pull

![image-20231101143041042](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101143041042.png)

![image-20231101143631566](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101143631566.png)

这里为了方便执行，直接选择启动则执行的，直接/etc/init.d下的启动执行的.sh的脚本

植入info.sh

直接输出得到结果：

![image-20231101154836994](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101154836994.png)

这里就可以直接通过adb pull 拉出相关的登录接口

![image-20231101155533117](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101155533117.png)



## 0x03 上帝视角

​		切换回上帝视角，这里直接通过咨询要到了adb shell密码，登录进去后发现其实就是这里的库文件控制的

![image-20231101161055850](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101161055850.png)

可以看到，quectel-ID作为关键的字符进行搜索，很快就可以定位到相应的位置，相应位置下的程序可以看到，这里的getpass就是读取密码，而接下来就是根据quectel-ID算出密码，因此，关键就是进入到sub_B9C进行加密处理

![image-20231101161622661](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101161622661.png)

这里为了确定调用点，可以通过自编写软件，直接遍历/proc找到对应的pid

![image-20231101163850814](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101163850814.png)

![image-20231101163909676](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101163909676.png)

直接可以定位到调用程序为/bin/login.shadow 为调用相应的pam_loginpw.so的二进制脚本程序

所以这里就可以直接分析该so中的sub_B9C加密流程

## 0x04 逆向分析

​		暂未进入到sub_B9C，我暂时将其理解为加密流程，给他命名为“en_cry”这样很容易分析出来

![image-20231101165019949](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101165019949.png)

​		这里根据传递分析可以看出来，就是通过SH_console_quectel作为key进行加密

​		因此，直接可以进入en_cry进行分析：

![image-20231101165317254](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101165317254.png)

这里就不需要深入了，直接通过该函数进行加密即可



## 0x05 脚本输出

​		根据该逆向分析结果，直接可以输出代码：

```
import crypt
import sys

id = sys.argv[1]

salt = "$1$"+id+"$"

cry = crypt.crypt("SH_console_quectel",salt)

result = cry[len(salt):]

print(result)
```

![image-20231101165919652](../../../../img/%E4%B8%9C%E8%BD%AF_Tbox%E5%88%86%E6%9E%90_ADB_shell_password.assets/image-20231101165919652.png)

​		