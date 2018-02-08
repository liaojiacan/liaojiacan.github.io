title: Java根据指定的country和lang格式化时间
author: Jiacan Liao
tags:
  - java
  - 业务开发
categories: []
date: 2017-10-16 21:18:00
---
> 服务端或者客户端在做一些多语言的时候可能会涉及到时间戳的格式化，不同的语言或者不同的国家的时间的表达格式可能不同。

```
public  String formatDate(Date date,String lang,String country){
    Locale locale = new Locale(lang,country,"");
    DateFormat dateFormat = DateFormat.getDateTimeInstance(DateFormat.DEFAULT,DateFormat.MEDIUM,locale);
    return dateFormat.format(date);
}
```