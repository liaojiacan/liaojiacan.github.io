title: Source Tree 配置支持gerrit review
author: Jiacan Liao
date: 2018-02-07 15:34:45
tags:
---
> source tree 可以支持自定义菜单，我们自定义一个菜单，来实现gerrit push review操作

1.先创建一个脚本，这里我叫 git_push_gerrit.sh
```
#!/bin/bash
# Push for gerrit review
# Created by Liaojiacan on 6.2.2017.
# Copyright (c) 2018 liaojiacan. All rights reserved.
branch=$(git symbolic-ref --short -q HEAD)
git push origin HEAD:refs/for/$branch
```
2.在SourceTree创建一个自定义操作

![upload successful](/images/pasted-0.png)
