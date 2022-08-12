# [首页](/blog/)

***

# gRPC

服务端实现接口并启动一个gRPC服务去处理客户端调用，客户端使用stub提供与服务器相同的方法，gRPC服务可以和stub在不同的环境中运行及通行。

## protobuf

默认使用protobuf（也可以是JSON），需要定义.proto文件描述数据结构，然后通过插件生成对应语言的代码。

```
syntax = "proto3";
```
如果不声明则默认为proto2。

```
message Person {
  string name = 1;
  int32 id = 2;
  bool has_ponycopter = 3;
}
```
1. 每个字段都需要指定一个唯一的index作为属性名，最小值为1，1-15只使用一字节（全部字段信息），16-2047使用两字节，无法使用19000-19999。
2. 添加【repeated】表示为数组（可以为任意个数），否则为单字段（可以不传）。
3. 生成的java代码会为每个message单独生成class文件同时包含builder类。

```
enum Corpus {
  CORPUS_UNSPECIFIED = 0;
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
}
```
枚举中必须定义0，且位于第一个。

```
import "myproject/other_protos.proto";
```
导入其他proto文件，以使用其定义的结构。

```
package foo.bar;
```
定义包名，名称重复。

```
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```
不支持对象，但可以通过这种方式实现。

```
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
```
会生成服务接口和stub。

```
option java_package = "com.example.foo"; // 使用的java包名
option java_outer_classname = "Ponycopter"; // 指定将所有类生成到指定的外部包装类中
option java_multiple_files = true; // 为每个结构单独生成类文件。
option optimize_for = CODE_SIZE; // 生成较小的类，通过以来公共类和反射代码实现序列化、转换等其他操作。
```