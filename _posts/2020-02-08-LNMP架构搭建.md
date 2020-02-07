---
layout:     post
title:      LNMP架构搭建
subtitle:   搭建PHP+mysq 在 linux 下的运行环境
date:       2020-02-08
author:     BY kexiaohei
header-img: img/LNMP.jpg
catalog: true
tags:
    - php
    - linux
    - LNMP
    - mysql
---
# 搭建PHP+mysq 在 linux 下的运行环境

## 1、LNMP架构

即Linux+nginx+mysql+php架构搭建

分别指的是**Linux系统**、**Web服务Nginx软件**、**MySQL数据库系统**、**PHP编程语言**

### 1.1 剖析

#### 1.1.1 nginx

Nginx在Kali系统自带有了，如果没有的话，在命令行下执行一条 apt install nginx 命令便可安装

/etc/nginx/配置文件夹**/usr/share/nginx/**默认显示页面路径/usr/lib/nginx/模块依赖路径/usr/sbin/nginx可执行文件

启动nginx的命令是

service nginx start

打开 /etc/nginx/nginx.conf, 可以发现访问日志和错误日志的路径分别是

```
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;
```

接着我们去查看访问日志

```
tail -f /var/log/nginx/access.log
```

然后浏览器刷新一下，看到了新访问记录，就可以证明现在运行的确是nginx

#### 1.1.2 Mysql

Kali也自带了mysql服务器。直接**service mysql start**就可以启动，默认没有密码

如果你的kali没有，那么可以运行命令 **apt install default-mysql-server** 进行安装

我们连接mysql看一下，运行命令

**mysql -u root**

可以看到直接进来了，不用输入密码

为了方便后续的使用，我们先给root账户设置一个密码，因为kali内置的mysql有些特殊，所以要用下面的命令来修改密码，这里我把密码修改为123456

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION;
flush privileges;
```

#### 1.1.3 PHP

kali只默认安装了php cli环境，并没有自带php-fpm组件，php-fpm是搭建web系统常用的一个组件，需要我们输入 **apt install php7.2-fpm**进行安装

在安装完毕后,只需要输入 **service php7.2-fpm start** 便可以启动了

和nginx一样，配置文件放在/etc/下，具体是**/etc/php/7.2/**

由于我们现在使用的是php的fpm模式，所以主要看fpm下的配置

分析出fpm本身的配置在 **/etc/php/7.2/fpm/pool.d/www.conf** 文件里

```
listen = /run/php/php7.2-fpm.sock
//这段配置说明fpm现在用的是以UNIX Domain Socket的形式在监听，其它应用可以通过这个文件和php-fpm通讯，这行配置需要记住
//还有一个要记住的配置是
user = www-data
group = www-data 
//这段配置是说明php-fpm是以用户组www-data的用户www-data运行的，这样做是为了限定php-fpm的权限，这样当php应用程序有漏洞的时候，对服务器的影响有所降低，起码不会是root级别的影响。
```

然后找一下日志相关的配置，在今后的使用中，我们可能经常需要打开日志，方便排错

\# access.log = log/$pool.access.log

access.log 是访问php-fpm组件的日志，默认是被注释的，也就是没有启用日志功能

这里我们把注释符号删掉，让它起作用

改完之后我们重启一下php的fpm组件



service php7.2-fpm restart



接下来我们让nginx和php的fpm组件进行通讯,这里可以参考php官方文档



https://secure.php.net/manual/zh/install.unix.nginx.php



我们来配置Nginx，我们找到和打开/etc/nginx/nginx.conf



可以看到有2行配置



include /etc/nginx/conf.d/*.conf;

include /etc/nginx/sites-enabled/*;



可以看到它的意思是分别是说

引入 /etc/nginx/conf.d/ 下所有扩展名是conf的文件

引入 /etc/nginx/sites-enabled/ 下所有文件



那么我们看一下 **/etc/nginx/conf.d/** 文件夹，发现没有东西可以参考。我们再去到**/etc/nginx/sites-enabled/**

发现有一个**default**文件，这个是nginx默认站点配置，我们看一看，然后在上面改



可以看到一行配置

**root /var/www/html;**

这行配置的意思是站点根目录在 /var/www/html文件夹下

我们在下面加多一行

**index index.php index.html;**

这句配置是默认去寻找index.php文件，找不到index.php就去找index.html

为什么是这个文件呢，因为大部分php项目都是用index.php文件作为一个入口的，只是习惯而已

往下翻,能看到有一段被注释的php相关的配置

```
#location ~ \.php$ {
\# include snippets/fastcgi-php.conf;
\#
\# # With php-fpm (or other unix sockets):
\# fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
\# # With php-cgi (or other tcp sockets):
\# fastcgi_pass 127.0.0.1:9000;

\# }
```

我们在下面把php官方文档的说明复制在下面,然后跟着改一下

```
location ~* \.php$ {

fastcgi_index index.php;

fastcgi_pass 127.0.0.1:9000;

include fastcgi_params;

fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

fastcgi_param SCRIPT_NAME $fastcgi_script_name;

}
```

这里把因为php的fpm组件现在是以socket文件通讯的，所以这里要修改的应该是

**fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;** 这一段

还记得刚刚fpm里面的listen配置么，把我们把sock文件路径改成fpm里的sock文件路径，也就是

fastcgi_pass unix:/run/php/php7.2-fpm.sock; 

然后执行 nginx -t 看一下配置有没有语法错误的地方

看到这里就应该知道配置没有语法错误,我们重启一下nginx

**service nginx restart**

接着我们怎么去验证nginx能和php通讯了呢?

我们去站点根目录创建一个php文件来测试

我们创建一个简单的php信息的页面，并保存为1.php

```php
<?php
phpinfo();
?>
```

接下来我们访问 http://127.0.0.1/1.php 看到了一个显示php各种信息的页面，说明我们的nginx已经成功和php进行通讯了