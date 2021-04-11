## Golang 与 redis 数据库的交互

1. 需要安装redis包
```bash
go get github.com/gomodule/redigo/redis
```

2. 创建测试文件,确保redis命令安装成功
```go
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)

func main() {
	conn, err :=redis.Dial("tcp","192.168.100.200:6379")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer conn.Close()

	conn.Do("auth","admin123")

	conn.Do("set","c1","hello")
}
```

在 redis 中 查看到代码中所创建爱的key成功

### 操作方法

Go操作redis文档:[https://godoc.org/github.com/gomodule/redigo/redis](https://godoc.org/github.com/gomodule/redigo/redis)

#### 连接数据库
```go
func Dial(network, address string, options ...DialOption) (Conn, error) {
	return DialContext(context.Background(), network, address, options...)
}

// 参数一: 连接的类型 tcp或者udp

// 参数二:ip地址和端口

// 返回连接对象和错误信息

```

#### 操作数据库
1. Send(commandName string, args ...interface{}) error
 send 函数用于将指定的 redis 命令放大缓冲区中,然后由  Flush 将缓冲区中的命令一次性刷新到服务器执行,但是使用Receive()函数接收返回值的时候,只保存第一次执行命令的结果
```go
// 参数一:命令
// 后面的参数是 key 和 value
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)

func main() {
	conn, err :=redis.Dial("tcp","172.19.36.69:6379")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer conn.Close()
  // 将命令放到缓冲区
    conn.Send("set","go","good")
  
	conn.Send("get","go")
    // 执行缓冲区的命令到服务器
	conn.Flush()
    // 获取执行的结果
	v ,err :=conn.Receive()
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(v)
}
```

2. 	Do(commandName string, args ...interface{}) (reply interface{}, err error)

- 参数 一: 要执行的命令
- 后面的参数:执行命令对应的参数
- 返回值为命令执行的结果和错误信息
```go
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)

func main() {
	conn, err :=redis.Dial("tcp","172.19.36.69:6379")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer conn.Close()
	conn.Do("auth", "admin123")
	rel , err := conn.Do("get","go")

	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(rel)
}
```

### 回复助手函数
Bool，Int，Bytes，map，String，Strings和Values函数将回复转换为特定类型的值。为了方便的包含对连接 Do 和 Receive 方法的调用,这些函数采用了类型为 error 的第二个参数,如果错误是非 nil,这复制函数返回错误,如果错误为 nil,则该函数将返回值转换为指定的类型
```bash
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)

func main() {
	conn, err :=redis.Dial("tcp","172.19.36.69:6379")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer conn.Close()
	conn.Do("auth", "admin123")
    // redis 存放的数据类型为字节码,转换为字符串类型
	rel , err := redis.String(conn.Do("get","go"))

	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(rel)
}
```

### scan

- func Values(reply interface{}, err error) (\[\]interface{}, error)
将返回值都转换为数组,然后存放在数组中

- func Scan(src \[\]interface{}, dest ...interface{}) (\[\]interface{}, error)
Scan 函数将数据从 src 复制到 dest 指向的值,首先 Scan 是在指定的数据中扫描,将扫描的值逐个的复制到后面给定的值中

```go
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)

func main() {
	conn, err :=redis.Dial("tcp","172.19.36.69:6379")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer conn.Close()
	conn.Do("auth", "admin123")
    // 执行命令,获取指定的数据,放到数组中
  rel ,err := redis.Values(conn.Do("mget","name","age","class"))
    
	if err != nil {
		fmt.Println(err)
		return
	}
    
	var name , class string
	var age int
    //  将数组中的值,赋值给指定的值
	_ ,err = redis.Scan(rel,&name,&age,&class)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(name,age,class)
}
```

### 序列化和反序列化
由于 Scan 或者回复助手函数只能将常见的类型复制给指定的数据,想我们自定义的结构体类型是不能被 Scan 处理的,那么久需要使用到序列化,在存储数据的时候,先把数据序列化后在存储,获取数据的时候,将获取到的数据反序列化

#### 序列化(字节化)
将指定的数据,转化成字节
```go
var buffer bytes.Buffer//容器
enc :=gob.NewEncoder(&buffer)//编码器,指定的是容器的地址,返回值是编码器对象
err:=enc.Encode(dest)//编码,参数是指定要编码的数据,返回值是错误信息

// 经过上面的编码之后,便可以将编码之后的 buffer 变量存储到 redis 中
// 编码之后的类型为字节类型
```

#### 反序列化(反字节化)

```go
dec := gob.NewDecoder(bytes.NewReader(buffer.bytes()))//解码器
dec.Decode(src)//解码,参数是要存放数据的字节数组
```


```bash
func main()  {
	var buffer bytes.Buffer

	var types string = "aaaa"

	enc :=gob.NewEncoder(&buffer)

	err := enc.Encode(&types)
	if err != nil {
		fmt.Println("1",err)
	}

	dec := gob.NewDecoder(bytes.NewReader(buffer.Bytes()))
	var aaa string
	dec.Decode(&aaa)
	fmt.Println(aaa)

}
```

### 序列化示例
例如:现在有自定义的结构体类型需要存储于展示
```go
package main

import (
	"bytes"
	"encoding/gob"
	"fmt"
	"github.com/gomodule/redigo/redis"
)

// 文章类别结构体
type ArticleType struct {
	Id       int
	TypeName string
}

func main() {
	conn, err :=redis.Dial("tcp","172.19.36.69:6379")
	if err != nil {
		fmt.Println("连接失败",err)
		return
	}
	defer conn.Close()
	_,err = conn.Do("auth","admin123")
	if err != nil {
		fmt.Println("连接redis",err)
		return
	}

	types := ArticleType{1,"体育新闻"}

	// 序列化
	var buffer bytes.Buffer  // 创建容器
	enc := gob.NewEncoder(&buffer)	// 编码器
	// 编码
	err = enc.Encode(types)
	if err != nil {
		fmt.Println("序列化失败",err)
		return
	}
	// 存储redis
	_, err = conn.Do("set","type",buffer.Bytes())
	if err != nil {
		fmt.Println("redis存储失败",err)
		return
	}

	// 读取数据
	rel , err := redis.Bytes(conn.Do("get","type"))
	if err != nil {
		fmt.Println("读取数据失败",err)
		return
	}
	// 解码器
	dec := gob.NewDecoder(bytes.NewReader(rel))
	// 创建用于保存数据的字节数组
	var col []ArticleType
	// 解码
	dec.Decode(&col)
	fmt.Println(col)

}
```


## go 语言连接redis 集群
安装需要的包
```bash
go get github.com/gitstliu/go-redis-cluster
```


### 操作集群
```go
package main

import (
	"fmt"
	"github.com/gitstliu/go-redis-cluster"
	"time"
)

func main() {
	cluster , _ := redis.NewCluster(&redis.Options{
		StartNodes: []string{"192.168.110.37:7000", "192.168.110.37:7001", "192.168.110.37:7002","192.168.110.38:7003","192.168.110.38:7004","192.168.110.38:7005"},
		ConnTimeout: 50 * time.Millisecond,// 连接时间
		ReadTimeout: 50 * time.Millisecond,// 读取时间
		WriteTimeout: 50 * time.Millisecond,//写入时间
		KeepAlive: 16,// 保持连接时间
		AliveTime: 60 * time.Second,// 生存时间
	})
	// 操作集群
	cluster.Do("set","name","itheima")

	name,_ := redis.String(cluster.Do("get","name"))
	fmt.Println(name)
}
```