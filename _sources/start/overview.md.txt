# 概述

翻译自：[overview](https://developers.google.cn/protocol-buffers/docs/overview)。

本指南描述了如何使用协议缓冲区语言来构造你的协议缓冲区数据，包括.proto文件的语法和如何从你的.proto文件生成数据访问类。它涵盖了协议缓冲区语言的proto2版本：关于proto3语法的信息，请参阅 [Proto3 语言指南](https://developers.google.cn/protocol-buffers/docs/proto3)。

这是一个参考指南——关于使用本文档中描述的许多功能的入门的例子，请参阅你所选择的语言的 [教程](https://developers.google.cn/protocol-buffers/docs/tutorials)。

## 定义消息类型

首先让我们看一个非常简单的例子。假设你想定义一个搜索请求的消息格式，每个搜索请求都有一个查询字符串，你感兴趣的特定结果页，以及每页的结果数量。这里是你用来定义消息类型的 `.proto` 文件：

```proto
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```

`SearchRequest` 消息定义指定了三个字段（名称/值对），每一个字段代表你想在这种类型的消息中包含的数据。每个字段都有一个名称和一个类型。

### 指定字段类型

在上面的例子中，所有的字段都是 [标量类型](#scalar)：两个整数（`page_number` 和 `result_per_page`）和一个字符串（`query`）。然而，你也可以为你的字段指定复合类型，包括枚举和其他消息类型。

(scalar)=
## 标量值类型

(enum)=
## 枚举