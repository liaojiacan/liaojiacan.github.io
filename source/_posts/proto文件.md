title: proto文件
author: Jiacan Liao
tags:
  - rpc
  - Protobuf
categories: []
date: 2017-10-10 19:52:00
---
> proto文件是Proto buffers的描述文件


- syntax 指定pd编译器的版本，可以设置proto2或者proto3
- message 类似java中的class关键字，在PB这里叫消息体
- service 服务声明
- 修饰符
  - required 非空，必须存在
  - optional 可选
  - repeated 可重复出现，类似集合的概念吧
- 更多介绍参考官方文档，[Protocal Buffers](https://developers.google.com/protocol-buffers/docs/proto3#simple)

```
syntax = "proto3";

service SearchService{
    rpc search(SearchRequest) returns (SearchResponse) {}
}

message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}

message SearchResponse {
  string result = 1;
}

```
