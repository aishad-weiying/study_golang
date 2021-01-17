## ProtoBuf 格式的文件的定义

```protobuf
syntax = "proto3"; // 指定proto版本
package hello;    // 指定默认包
// 定义go_package 包
option go_package = ".;hello";

// 定义hello 服务
service Hello {
  // 定义sayhello方法
  rpc SayHello(HelloRequest) returns (HelloResponse){}
}
// 定义HelloRequest 请求结构
message HelloRequest {
  string name = 1;
}

// 定义HelloResponse 结构体
message  HelloResponse {
  string resp = 1;
}
```

1. package

在proto文件中,使用package关键词声明包名,默认转换成go中的包名与此一致,如果需要制定不一样的包名,使用`go_package`选项

```protobuf
option go_package = ".;hello";
```

2. message

proto中的message对于应go中的结构体,全部使用驼峰式命名规则,嵌套定义的message,enum转换成go之后,名称变为Parent_child 的结构体

```protobuf
// Test 测试
message Test {
    int32 age = 1;
    int64 count = 2;
    double money = 3;
    float score = 4;
    string name = 5;
    bool fat = 6;
    bytes char = 7;
    // Status 枚举状态
    enum Status {
        OK = 0;
        FAIL = 1;
    }
    Status status = 8;
    // Child 子结构
    message Child {
        string sex = 1;
    }
    Child child = 9;
    map<string, string> dict = 10;
}
```

转换成go后的结构

```go
// Status 枚举状态
type Test_Status int32

const (
    Test_OK   Test_Status = 0
    Test_FAIL Test_Status = 1
)

// Test 测试
type Test struct {
    Age    int32       `protobuf:"varint,1,opt,name=age" json:"age,omitempty"`
    Count  int64       `protobuf:"varint,2,opt,name=count" json:"count,omitempty"`
    Money  float64     `protobuf:"fixed64,3,opt,name=money" json:"money,omitempty"`
    Score  float32     `protobuf:"fixed32,4,opt,name=score" json:"score,omitempty"`
    Name   string      `protobuf:"bytes,5,opt,name=name" json:"name,omitempty"`
    Fat    bool        `protobuf:"varint,6,opt,name=fat" json:"fat,omitempty"`
    Char   []byte      `protobuf:"bytes,7,opt,name=char,proto3" json:"char,omitempty"`
    Status Test_Status `protobuf:"varint,8,opt,name=status,enum=test.Test_Status" json:"status,omitempty"`
    Child  *Test_Child `protobuf:"bytes,9,opt,name=child" json:"child,omitempty"`
    Dict   map[string]string `protobuf:"bytes,10,rep,name=dict" json:"dict,omitempty" protobuf_key:"bytes,1,opt,name=key" protobuf_val:"bytes,2,opt,name=value"`
}

// Child 子结构
type Test_Child struct {
    Sex string `protobuf:"bytes,1,opt,name=sex" json:"sex,omitempty"`
}
```

除了会生成对应的结构之外,还会有些工具方法,比如字段的getter

```go
func (m *Test) GetAge() int32 {
    if m != nil {
        return m.Age
    }
    return 0
}
```

枚举类型会生成对应名称的常量,同时会有两个map方便使用

```go
var Test_Status_name = map[int32]string{
    0: "OK",
    1: "FAIL",
}
var Test_Status_value = map[string]int32{
    "OK":   0,
    "FAIL": 1,
}
```

3. service

定义一个简单的service,比如上面的TestService有一个方法叫做Test,接收一个`Request`类型的参数,返回值为`Response`类型的参数

```protobuf
// TestService 测试服务
service TestService {
    // Test 测试方法
    rpc Test(Request) returns (Response) {};
}

// Request 请求结构
message Request {
    string name = 1;
}

// Response 响应结构
message Response {
    string message = 1;
}
```

转换成go语言之后的结构为

```go
// 客户端接口
type TestServiceClient interface {
    // Test 测试方法
    Test(ctx context.Context, in *Request, opts ...grpc.CallOption) (*Response, error)
}

// 服务端接口
type TestServiceServer interface {
    // Test 测试方法
    Test(context.Context, *Request) (*Response, error)
}
```

### proto 语法

1. 基本规范

- 文件以`.proto`作为后缀,除结构定义外的语句,以分号结尾
- 结构定义可以包含: messaeg,service,enum
- rpc 方法定义结尾的分号可有可无
- Message类型采用驼峰式命名方式,字段名采用小写字母加上下划线分割的方式
- Enum 类型采用驼峰式命名方式,字段命名采用的是大写字母加上下划线分割的方式
- service 与rpc方法名统一采用驼峰式命名

2. 字段规则

- 字段格式 : 限定修饰符 | 数据类型 | 字段名称 | = | 字段编码值 | [字段默认值]
- 限定修饰符包括 : required| optional| repeated
  - Required: 表示是一个必须字段,必须相对于发送发,在发送消息之前必须设置这个字段的值,对于接收方,必须能够识别该字段的意思,发送之前没有设置required字段或者无法识别这个字段都会引发编译异常,导致消息被丢弃
  - Optional : 表示是一个可选字段,可选对于发送方,在消息发送的时候,可以选择性的设置或者不设置这个字段,对于接收方来说,如果能够识别可选字段的值就进行相应的处理,如果无法识别,就忽略该字段,消息中的其他字段正常处理
  - Repeated: 该字段表示可以包含0~n个元素,其特性和optional一样,但是每一次可以包含多个值,可以看做是传递一个数组的值
- 数据类型
  - protoBuf定义了一套基本数据类型,几乎都可以映射到各个语言的基本数据类型

| .proto   | C++    | Java       | Python         | Go      | Ruby                 | C#         |
| -------- | ------ | ---------- | -------------- | ------- | -------------------- | ---------- |
| double   | double | double     | float          | float64 | Float                | double     |
| float    | float  | float      | float          | float32 | Float                | float      |
| int32    | int32  | int        | int            | int32   | Fixnum or Bignum     | int        |
| int64    | int64  | long       | ing/long[3]    | int64   | Bignum               | long       |
| uint32   | uint32 | int[1]     | int/long[3]    | uint32  | Fixnum or Bignum     | uint       |
| uint64   | uint64 | long[1]    | int/long[3]    | uint64  | Bignum               | ulong      |
| sint32   | int32  | int        | intj           | int32   | Fixnum or Bignum     | int        |
| sint64   | int64  | long       | int/long[3]    | int64   | Bignum               | long       |
| fixed32  | uint32 | int[1]     | int            | uint32  | Fixnum or Bignum     | uint       |
| fixed64  | uint64 | long[1]    | int/long[3]    | uint64  | Bignum               | ulong      |
| sfixed32 | int32  | int        | int            | int32   | Fixnum or Bignum     | int        |
| sfixed64 | int64  | long       | int/long[3]    | int64   | Bignum               | long       |
| bool     | bool   | boolean    | boolean        | bool    | TrueClass/FalseClass | bool       |
| string   | string | String     | str/unicode[4] | string  | String(UTF-8)        | string     |
| bytes    | string | ByteString | str            | []byte  | String(ASCII-8BIT)   | ByteString |

```
+ N 表示打包的字节并不是固定。而是根据数据的大小或者长度
+ 关于 fixed32 和int32的区别。fixed32的打包效率比int32的效率高，但是使用的空间一般比int32多。因此一个属于时间效率高，一个属于空间效率高
```

- 字段名称
  - 字段名称的命名与其他语言的变量命名方式类似,protobuf建议字段的命名采用以下划线分割的驼峰式
- 字段编码值
  - 有了字段的编码值,通信的双发才能互相识别对方的字段,相同的编码值,其限定修饰符和数据类型必须相同,编码值的取值范围是`1~2^32`
  - 其中1~15的编码时间和空间的效率是最高的,编码值越大,其编码的时间和空间效率就越低,所以建议把经常要传递的值的字段编码值设置在1~15之间
- 字段默认值
  - 在数据的传递的时候,对于Required类型的户数,如果用户没有设置值,则使用默认值传递到对端

### service的定义

如果想要将消息类型用在RPC系统中,可以在`.proto`文件中定义一个RPC服务接口,protobuf编译器会根据所选择的不同语言生成不用的服务结构代码



例如: 想要定义一个RPC服务并具有一个方法,这个方法接收SearchRequest并返回SearchResponse

```go
service SearchService {
    rpc Search (SearchRequest) returns (SearchResponse){}
}
```

生成的接口代码作为客户端与服务端的约定,服务端必须实现定义的所有接口方法,客户端直接调用同名方法向服务端发起请求,比较麻烦的是,即便是业务上不需要参数也必须指定一个请求消息,一般会定义一个空的message

### message 的定义

一个message类型定义描述了一个请求或者响应的消息格式,可以包含多种类型字段,例如: 定义一个搜索请求的消息格式,每个请求包含查询字符串,页码,每页数目等

字段名用小写,转为go文件后自动变为大写,message就相当于结构体

```go
syntax = "proto3";
message SerachRequest {
    string query = 1;			// 查询的字符串
    int32 page_number = 2;		// 页码
    int32 result_per_page = 3:	// 每页的条目数
}
```

首行声明使用的protobuf版本为proto3

SearchRequest 定义了三个字段



#### 嵌套使用message

```protobuf
 message SearchResponse {
        repeated Result results = 1;
    }

    message Result {
        string url = 1;
        string title = 2;
        repeated string snippets = 3;
    }
```

内部声明的message只能在内部使用

```protobuf
 message SearchResponse {
        message Result {
            string url = 1;
            string title = 2;
            repeated string snippets = 3;
        }
        repeated Result results = 1;
    }
```

#### proto 的map 类型

```protobuf
    map<key_type, value_type> map_field = N;

    message Project {...}
    map<string, Project> projects = 1;
```

- 键或者值可以是内置类型,也可以是自定的message类型
- 字段不支持使用repeated等类型

### proto 文件的编译

- 通过定义好的.proto文件生成Java, Python, C++, Go, Ruby, JavaNano, Objective-C, or C# 代码，需要安装编译器protoc
- 当使用protocol buffer编译器运行.proto文件时，编译器将生成所选语言的代码，用于使用在.proto文件中定义的消息类型、服务接口约定等。不同语言生成的代码格式不同：
  - C++: 每个.proto文件生成一个.h文件和一个.cc文件，每个消息类型对应一个类
  - Java: 生成一个.java文件，同样每个消息对应一个类，同时还有一个特殊的Builder类用于创建消息接口
  - Python: 姿势不太一样，每个.proto文件中的消息类型生成一个含有静态描述符的模块，该模块与一个元类metaclass在运行时创建需要的Python数据访问类
  - Go: 生成一个.pb.go文件，每个消息类型对应一个结构体
  - Ruby: 生成一个.rb文件的Ruby模块，包含所有消息类型
  - JavaNano: 类似Java，但不包含Builder类
  - Objective-C: 每个.proto文件生成一个pbobjc.h和一个pbobjc.m文件
  - C#: 生成.cs文件包含，每个消息类型对应一个类

### import 导入定义

- 可以使用import语句导入使用其他描述文件中声明的类型
- protobuf 接口文件可以像C语言的h文件一个，分离为多个，在需要的时候通过 import导入需要对文件。其行为和C语言的#include或者java的import的行为大致相同，例如import "others.proto";
- protocol buffer编译器会在 -I / --proto_path参数指定的目录中查找导入的文件，如果没有指定该参数，默认在当前目录中查找

在其他proto文件的message中可以使用其他文件定义的message,通过`包名+消息名`的方式来使用

- 在.proto文件中使用package声明包名，避免命名冲突

```
syntax = "proto3";
package foo.bar;
message Open {...}
```

- 在其他的消息格式定义中可以使用包名+消息名的方式来使用类型，如

```
message Foo {
    ...
    foo.bar.Open open = 1;
    ...
}
```

- 在不同的语言中，包名定义对编译后生成的代码的影响不同
  - C++ 中：对应C++命名空间，例如Open会在命名空间foo::bar中
  - Java 中：package会作为Java包名，除非指定了option jave_package选项
  - Python 中：package被忽略
  - Go 中：默认使用package名作为包名，除非指定了option go_package选项
  - JavaNano 中：同Java
  - C# 中：package会转换为驼峰式命名空间，如Foo.Bar,除非指定了option csharp_namespace选项

## 小案例

编写一个hello项目,项目定义了一个Hello Service,客户端发送任意的字符串,服务器端返回hello消息

流程:

- 编写proto文件
- 编译生成.pb.go文件
- 服务端实现约定好的接口,并提供服务
- 客户端使用.pb.go文件中的方法去请求服务

项目结构

```
|—— hello/
    |—— client/
        |—— main.go   // 客户端
    |—— server/
        |—— main.go   // 服务端
|—— proto/
    |—— hello/
        |—— hello.proto   // proto描述文件
        |—— hello.pb.go   // proto编译后文件
```

1. 编写proto文件

```protobuf
syntax = "proto3"; // 指定proto版本
package hello;    // 指定默认包
// 定义go_package 包
option go_package = ".;hello";

// 定义hello 服务
service Hello {
  // 定义sayhello方法
  rpc SayHello(HelloRequest) returns (HelloResponse){}
}
// 定义HelloRequest 请求结构
message HelloRequest {
  string name = 1;
}

// 定义HelloResponse 结构体
message  HelloResponse {
  string resp = 1;
}
```

2. 编译生成.pb.go文件

```bash
$ cd proto/hello

# 编译hello.proto
$ protoc -I . --go_out=plugins=grpc:. ./hello.proto
```

3. 服务端根据文件实现定义的接口

```go
package main

import (
	"fmt"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/grpclog"
	pb "grpc_test/proto/hello"
	"net"
)

const Address = "127.0.0.1:50052"

// 定义HelloService并实现其约定的接口
type helloService struct {}
// HelloService Hello服务
var HelloService = helloService{}

// 实现sayhello的接口
func (h helloService) SayHello(ctx context.Context,in *pb.HelloRequest)(*pb.HelloResponse,error) {
	resp := new(pb.HelloResponse)
	// 定义返回的数据
	resp.Resp = fmt.Sprintf("Hello %s",in.Name)
	return resp,nil
}

func main()  {
	listen,err := net.Listen("tcp",Address)
	if err != nil{
		grpclog.Fatalf("failed to listen:%v",err)
	}
	// 实例化grpc server
	s := grpc.NewServer()
	// 注册helloserver
	pb.RegisterHelloServer(s,HelloService)
	fmt.Println("listen on "+ Address)
	s.Serve(listen)
}

```

4. 客户端调用

```go
package main

import (
	"fmt"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/grpclog"
	"grpc_test/proto/hello"
)

// 服务端的地址
const Address = "127.0.0.1:50052"

func main() {
	conn,err := grpc.Dial(Address,grpc.WithInsecure())
	if err != nil{
		grpclog.Fatalln(err)
	}
	defer conn.Close()
	// 初始化客户端
	c := hello.NewHelloClient(conn)
	// 调用方法
	req := &hello.HelloRequest{
		Name: "grpc",
	}
	resp ,err := c.SayHello(context.Background(),req)
	if err != nil {
		grpclog.Fatalln(err)
	}
	fmt.Println(resp.Resp)
}

```

