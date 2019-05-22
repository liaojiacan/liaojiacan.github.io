title: Idea使用骨架生成项目卡住的解决办法
author: Jiacan Liao
tags:
  - Idea
  - archetypeCatalog
categories:
  - 工具使用
date: 2018-03-09 19:01:00
---
&emsp;&emsp;Idea在使用骨架生成项目时（Create from archetype）有时候会发现卡住了，控制台停留在

```
[INFO] Generating project in Batch mode
```

通过debug日志发现是搜索archetype-catalog.xml卡住了

```
[INFO] Generating project in Batch mode
[DEBUG] Searching for remote catalog: http://repo.maven.apache.org/maven2/archetype-catalog.xml
```
解决方法：

&emsp;&emsp; 修改archetypeCatalog参数，archetypeCatalog=internal。Idea可以在maven的runner配置中指定。如图：

![upload successful](/images/pasted-2.png)

archetypeCatalog参数取值可以从maven的文档查到.[Generate project using an alternative catalog
](http://maven.apache.org/archetype/maven-archetype-plugin/examples/generate-alternative-catalog.html)

archetypeCatalog 可配置的值有

- internal to use the internal catalog only.
- local to use the local catalog only.
- remote to use the maven's remote catalog.

```
No catalog is currently provided.
The default value is remote,local. Thus the local catalog is shown just after the remote one.
```

默认是remote,local. 这里解决方法其实有两个，
一个就是上面所说修改archetypeCatalog=internal,直接使用网络。另外一个就是修改远程仓库，如使用阿里的镜像。

