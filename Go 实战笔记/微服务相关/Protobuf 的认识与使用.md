### 一、什么是 protobuf
> Protobuf 是谷歌内部用来实现对混合语言数据的一种序列化结构数据的方法，它的作用就是将结构化的数据进行序列化，可用于通信协议、数据存储等。

简而言之，用它更节省资源。形象举例：
在 PHP 里面我定义了一个数组，现在我通过 protobuf 将其序列化后，我在 Go 里面反序列化，然后在 Go 里面我也能认识和处理这段数据。有点类似 json，而 Protobuf 更快、更小。

### 二、protobuf 高效的原因
1、不受语言和平台的限制  
Protobuf 支持 Java、C++、Python、PHP、Go 等多种语言，支持多个平台。

2、高效  
他比 XML 更小（3 ~ 10倍）、更快（10 ~ 100倍）、更简单

3、拓展性、兼容性  
更新数据结构，不会影响和破坏原因程序

### 三、如何使用
protobuf 已经更新到 `proto3`，下面都是将的 `proto3` 相关内容。
#### 3.1 工作原理
![protobuf 工作原理图](https://developers.google.com/static/protocol-buffers/docs/images/protocol-buffers-concepts.png)

#### 3.2 定义数据结构
Protobuf 的数据结构原文件的拓展名是 `.proto`，我们先定义如下结构：
```protobuf
syntax = "proto3";

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```
协议缓冲区支持通常的原始数据类型，例如整数、布尔值和浮点数，当然一个字段也可以是：

`message` 类型，以便您可以嵌套部分定义，例如用于重复数据集。  
`enum` 类型，因此您可以指定一组值以供选择。  
`oneof` 类型，当消息有许多可选字段并且最多同时设置一个字段时可以使用该类型。  
`map` 类型，用于将键值对添加到您的定义中。  

通过 `syntax = "proto3"；` 在第一个非空、非注释行来指定我们使用的是 `proto3` 语法，否则将被认为是 `proto2`。

**字段编号**
上面的定义中 `query` 的编号是 1，`page_number` 是 2。在 `proto3` 中，`1 ~ 15` 一般用来放频繁出现的字段，因为其只占一个字节码，`16 ~ 2047` 范围内的字段编号需要两个字节码，在定义时，我们可以为以后需要频繁出现的字段元素留一定的空间。大致范围如下：
最小是 1，最大是 2<sup>29 - 1</sup>，即 536,870,911。`19000 ~ 19999` 是 Protobuf 内部保留的不可以使用。

**保留字段**
保留字段将不能被使用，比如我们将某个字段移除，而不是直接删除或者注释掉，这可能会导致数据的损坏，最好的办法就是将其编号或者字段名设置为保留字段。例如：
```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
注意：字段编号和字段名不能混合一行写。

[proto3 更多关于字段的内容](https://developers.google.com/protocol-buffers/docs/proto3)

#### 3.3 使用
我们定义好消息结构后，我们需要使用协议缓存编译器生成对应语言所支持的消息结构，方便序列化转换。在 Go 语言中通常生成 `.pb.go` 的文件。 