## 结构体和 JSON 序列化

JSON(Javascript Object Notation),是一种轻量级的数据交换格式,方便阅读和编写,同时也易于机器解析和生成,JSON 键值对是用来保存 JS 对象的一种方式,键值对组合中,键名写在前面并使用双引号包裹,键值之间使用冒号分割,冒号后面是对应的值,多个键值对使用逗号分割

#### 为什么要使用 json
golang 的程序要想与别的程序或者接口通信的话,那么就需要有一个中间的格式,来进行消息传递,那么 json 就是起到这样的作用,golang 将数据转换成 json 格式以后,通过网络进行传输,发送给其他程序后,进行反序列化得到自己支持的语言格式

### 将结构体转化为json 格式

使用的方法为 json.Marshal
```go
func Marshal(v interface{}) ([]byte, error) 
```

参数为接口类型,返回值为字符切片和错误信息

```go
package main

import (
	"encoding/json"
	log "github.com/sirupsen/logrus"
)

type Students struct {
	Id int
	Name string
	Age int
}

func main()  {
	var stu1 = Students{
		Id:10,
		Name:"wang",
		Age:20,
	}
	res ,err := json.Marshal(stu1)
	if err != nil {
		log.Println("序列化错误:",err)
	}
	log.Println(string(res))

}
```

上面的格式中，序列化以后的 json 格式的数据都是在一行的，不便于阅读，我们可以使用json.MarshalIndent进行序列化
```go
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error) 

// 参数 1: 要序列化的数据
// 参数 2: 表示每一行输出的前缀
// 参数 3: 表示每一个层级的缩进

// 上面的代码可以写成如下:
res2 , err := json.MarshalIndent(stu1, "", "    ")
	if err != nil {
		log.Println("序列化错误:", err)
	}
	log.Println(string(res2))
```

> 在序列化的时候,只有导出的成员(结构体成员首字母大写)才会被编码

### 将 json 格式的数据转换成编程语言里面的对象

反序列化使用的包是json.Unmarshal
```go
func Unmarshal(data []byte, v interface{}) error
```
参数是字符切片和接口类型,字符切片是 json 格式的数据,转换完毕的数据存储在接口类型的数据汇总

```go
package main

import (
	"encoding/json"
	"fmt"

	log "github.com/sirupsen/logrus"
)

type Students struct {
	Id   int
	Name string
	Age  int
}

func main() {
	var stu1 = Students{
		Id:   10,
		Name: "wang",
		Age:  20,
	}
	res, err := json.Marshal(stu1)
	if err != nil {
		log.Println("序列化错误:", err)
	}
	log.Println(string(res))
	var stu2 Students
	str := "{\"Id\":10,\"Name\":\"wang\",\"Age\":20}"
	err = json.Unmarshal([]byte(str),&stu2)
	if err != nil {
		log.Println("反序列化失败:", err)
	}
	fmt.Println(stu2)
}
```

> 注意: 上面的结构体中,结构体元素的首字母必须大写,这样才能通过 json 给前端传值,否则,前端中获取到的 json 格式的数据是没有数据的

上面的方法中,如果首字母大写的话,序列化以后的数据,key 值也是大写的,如果要想序列化后的数据时小写的话,使用下面的方法
```go
// Students 结构体
type Students struct {
	Sid  int    `json:"id"`
	Name string `json:"name"`
	Age  int    `json:"age"`
}
```