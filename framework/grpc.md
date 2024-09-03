# [首页](/blog/)

***

# gRPC

服务端实现接口并启动一个gRPC服务去处理客户端调用，客户端使用stub提供与服务器相同的方法，gRPC服务可以和stub在不同的环境中运行及通行。

Grpc基于Http2，使用netty进行通行，即服务端启动netty服务器，客户端向服务端发送Http2请求。

## protobuf

默认使用protobuf（也可以是JSON），需要定义.proto文件描述数据结构，然后通过插件生成对应语言的代码。

```txt
syntax = "proto3"; // 如果不声明则默认为proto2。

import "myproject/other_protos.proto"; // 导入其他proto文件，以使用其定义的结构。

package foo.bar; // 定义proto文件的包名

option java_package = "com.foo.bar"; // 定义生成的java类的包名，不指定则默认为package。
option java_outer_classname = "FooBar"; // 指定外部包装类的类型，默认使用proto文件名转为大驼峰，该类必定会生成。
option java_multiple_files = true; // 为每个结构单独生产类，默认为false，将所有类生成为包装类的内部类。
option optimize_for = CODE_SIZE; // 生成较小的类，通过以来公共类和反射代码实现序列化、转换等其他操作。

// 1. 每个字段都需要指定一个唯一的index作为属性名，最小值为1，1-15只使用一字节（全部字段信息），16-2047使用两字节，无法使用19000-19999。
// 2. 添加【repeated】表示为数组（可以为任意个数），否则为单字段（可以不传）。
// 3. 生成的java代码会为每个message单独生成class文件同时包含builder类。
// 4. 注意没有包装类型，所有属性都会设置默认值，需要注意处理默认值。
message Person {
  string name = 1;
  int32 id = 2;
  bool has_ponycopter = 3;
}

// 不支持对象，但可以通过这种方式实现。
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;

enum Corpus {
  CORPUS_UNSPECIFIED = 0; // 枚举中必须定义0，且位于第一个。
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
}

// 会生成服务接口和stub。
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
```

## java code

```java
public class GreeterService extends GreeterImplBase {
    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        responseObserver.onNext(HelloReply.newBuilder().build()); // 发送消息
        responseObserver.onCompleted(); // 结束传送
    }
}
```
创建服务端实现类

```java
Server server = ServerBuilder.forPort(port).addService(new GreeterService()).build();
server.start();
```
启动服务，监听端口。

```java
var ManagedChannel = ManagedChannelBuilder.forAddress(host, port).usePlaintext().build();
```
创建客户端通道

```java
GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
HelloReply reply = stub.sayHello(HelloRequest.newBuilder().build());
```
创建阻塞型stub后，使用stub发起请求。

## use JSON

***

## maven

```xml
<build> <!--necessary for grpc compile-->
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.6.2</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.21.1:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.49.0:exe:${os.detected.classifier}</pluginArtifact>
                <!--指定以下配置将生成的文件输出到源码目录中，默认只会生成到jar包中-->
                <!--指定.proto文件根目录，默认${project.basedir}/src/main/proto-->
                <protoSourceRoot>${project.basedir}/src/main/resources/proto</protoSourceRoot>
                <!--指定.java文件生成目录，默认${project.basedir}/src/main/java-->
                <outputDirectory>${project.basedir}/src/main/java</outputDirectory>
                <!--设置是否清空output目录，即是否保留生成的.java文件，默认true。-->
                <clearOutputDirectory>false</clearOutputDirectory>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

***