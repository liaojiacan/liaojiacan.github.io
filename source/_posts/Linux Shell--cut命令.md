---
title: Linux shell命令:cut命令
date: 2017-02-23 16:45:44
categories: 
    - linux
tags: 
    - linux
    - shell
---

#### 简介
将一段数据经过分析，取出我们想要的

```
     cut -- cut out selected portions of each line of a file SYNOPSIS
     cut -b list [-n] [file ...]
     cut -c list [file ...]
     cut -f list [-d delim] [-s] [file ...]
```

- -b ：以字节为单位进行分割。这些字节位置将忽略多字节字符边界，除非也指定了 -n 标志。
- -c ：以字符为单位进行分割。
- -d ：自定义分隔符，默认为制表符。
- -f ：与-d一起使用，指定显示哪个区域。
- -n ：取消分割多字节字符。仅和 -b 标志一起使用。如果字符的最后一个字节落在由 -b 标志的 List 参数指示的范围之内，该字符将被写出；否则，该字符将被排除。


```
 who | cut -d 3-5 #截取 3-5列的内容
```
```
 who | cut -c 3-5 #截取 3-5列的内容 中文字符
```

```
 cat /etc/passwd|head -n 5|cut -d : -f -2 # 设置分隔符为：
```