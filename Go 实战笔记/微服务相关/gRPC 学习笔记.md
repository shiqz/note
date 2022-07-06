在 gRPC 中，客户端应用程序可以直接调用不同机器上的服务器应用程序上的方法，就像它是本地对象一样，使您更容易创建分布式应用程序和服务。与许多 RPC 系统一样，gRPC 基于定义服务的思想，指定可以远程调用的方法及其参数和返回类型。在服务端，服务端实现这个接口并运行一个 gRPC 服务器来处理客户端调用。在客户端，客户端有一个存根（在某些语言中仅称为客户端），它提供与服务器相同的方法。
![gRPC 原理图](https://grpc.io/img/landing-2.svg)

gRPC 中传输数据格式默认使用的是 Protobuf3，例如：
```protobuf
// The greeter service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

### 一、如何在 Go 中实现 gRPC
1、定义消息结构，即创建 `proto` 文件
2、Go 安装协议编译器插件
```bash
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
```
3、暴露环境变量
```bash
vim ~/.bash_profile

# 添加
# export GO_PATH=~/go
# export PATH=$PATH:/$GO_PATH/bin
source ~/.bash_profile
```
4、使用缓冲编译器编译成对应的代码
```bash
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    routeguide/route_guide.proto
```
5、官方案例
```bash
git clone -b v1.46.0 --depth 1 https://github.com/grpc/grpc-go
```