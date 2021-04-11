#### 首先根据之前的代码创建客户端与服务器

服务端: 将可读端发送的数据转化为大写,之后发送给客户端
```go
package main

import (
	"fmt"
	"log"
	"net"
	"strings"
)

func main() {
	// 1. 建立监听连接
	listerer, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		fmt.Println("建立监听失败:", err)
		return
	}
	defer listerer.Close()

	//2. 监听客户端连接
	for {
		conn, err := listerer.Accept()
		if err != nil {
			fmt.Println("监听客户端连接失败:", err)
			return
		}
		log.Println(conn.RemoteAddr(), "已经连接")
		go handlerconn(conn)
	}

}

func handlerconn(conn net.Conn) {
	defer conn.Close()
	// 接收客户端数据
	buf := make([]byte, 1024)
	for {
		n, err := conn.Read(buf)
		if n == 0 {
			fmt.Println("客户端", conn.RemoteAddr(), "已经关闭，服务器端断开连接")
			return
		}
		if err != nil {
			fmt.Println("读取客户端数据失败:", err)
			return
		}
		log.Println("读取到的数据为:", string(buf[:n]))
		str := string(buf[:n])
		if str == "exit\n" || str == "quit\n" || str == "exit\r\n" || str == "quit\r\n" {
			fmt.Println("客户端", conn.RemoteAddr(), "已经关闭，服务器端断开连接")
			return // 或者使用 runtime.Goexit()
		}
		// 发送数据给客户端
		conn.Write([]byte(strings.ToUpper(string(buf[:n]))))
	}

}
```

客户端: 向服务端循环发送 20 次how are you
```go
package main

import (
	"fmt"
	"log"
	"net"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:8080")
	if err != nil {
		log.Println("连接服务端失败:", err)
		return
	}
	defer conn.Close()
	go connServer(conn)

	for {
		// 创建缓冲区用于保存从服务器读取的数据
		buf := make([]byte, 4096)
		n, err := conn.Read(buf)
		if n == 0 { // 判断如果读取的数据长度为 0 ,说明服务器端已经关闭了连接,那么客户端也需要关闭连接
			fmt.Println("断开连接")
			return
		}
		if err != nil {
			fmt.Println("从服务器读取数据失败：", err)
			return
		}
		fmt.Println("从服务器读取的数据为：", string(buf[:n]))
	}
}

func connServer(conn net.Conn) {

	for i := 0; i < 20; i++ {
		// 给服务器发送数据
		_, err2 := conn.Write([]byte("how are you?"))
		if err2 != nil {
			fmt.Println("向服务器发送数据失败：", err2)
		}
	}
}
```

执行上面的代码之后,理论上应该是服务器接收到 20 次的`how are you`,然后每次将其转换为大写后,发送给客户端 20 次,下面我们来看一下实际的执行情况

服务器端
```go
2020/07/29 13:53:46 127.0.0.1:56085 已经连接
2020/07/29 13:53:46 读取到的数据为: how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?how are you?
```

客户端
```go
从服务器读取的数据为： HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?HOW ARE YOU?
```

从上面的代码的执行结果中,我们可以看出,无论是服务端还是客户端,都是只接收到了一次的数据,而这一次的数据中却包含了 20 次循环的数据,为什么会出现这种问题???

## TCP 黏包
上面的情况出现,我们成为 TCP 黏包

出现黏包的主要原因就是,因为 tco 数据的传递模式是流模式,在保持长连接的时候,可以进行多次的接收和发送

1. 由 Nagle算法造成的 tcp 黏包,nagle 算法是一种改善网络传输效率的算法,简单的来说,就是当我们提交一段数据给 TCP 发送的时候,TCP 并不是立即发送出去,而是等待一小段时间,看看这段时间内是否还有数据要发送,如果有,就把这两段数据一次性发送出去
2. 黏包既可以发生在发送端也可以发生在接收端,上面的案例中,黏包就是发生在发送端,如果接收端接收数据不及时的情况下,也会造成黏包:TCP 会把接收到的数据存在自己的缓冲区中,然后通知应用层程序取数据,当应用层由于某些原因不能及时把数据取出来的时候,就会造成 TCP 缓冲区中存放了几段数据

## 解决黏包
出现`黏包`的关键在于接收方不确定要传输的数据的大小,因此我们可以对数据包进行封包和拆包的操作

1. 封包
封包就是给一段数据加上包头,这样一来,数据包就分为包头和包体两部分内容了(过滤非法包时封包会加入`包尾内容`),包头部分的长度是固定的,并且存储了包体的长度,根据包头长度固定以及包头中包含包体的长度的变量就能正确的拆分出一个完整的包
```go
package proto

import (
	"bufio"
	"bytes"
	"encoding/binary"
)

// Encode 将消息编码
func Encode(message string) ([]byte, error) {
	// 读取消息的长度，转换成int32类型（占4个字节）
	var length = int32(len(message))
	var pkg = new(bytes.Buffer)
	// 写入消息头
	//以小端的方式写,大端的小端可百度查询
	err := binary.Write(pkg, binary.LittleEndian, length)
	if err != nil {
		return nil, err
	}
	// 写入消息实体
	err = binary.Write(pkg, binary.LittleEndian, []byte(message))
	if err != nil {
		return nil, err
	}
	return pkg.Bytes(), nil
}

// Decode 解码消息
func Decode(reader *bufio.Reader) (string, error) {
	// 读取消息的长度
	lengthByte, _ := reader.Peek(4) // 读取前4个字节的数据
	lengthBuff := bytes.NewBuffer(lengthByte)
	var length int32
	err := binary.Read(lengthBuff, binary.LittleEndian, &length)
	if err != nil {
		return "", err
	}
	// Buffered返回缓冲中现有的可读取的字节数。
	if int32(reader.Buffered()) < length+4 {
		return "", err
	}

	// 读取真正的消息数据
	pack := make([]byte, int(4+length))
	_, err = reader.Read(pack)
	if err != nil {
		return "", err
	}
	return string(pack[4:]), nil
}
```

服务端的代码
```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"proto"
	"strings"
)

func main() {
	// 1. 建立监听连接
	listerer, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		fmt.Println("建立监听失败:", err)
		return
	}
	defer listerer.Close()

	//2. 监听客户端连接
	for {
		conn, err := listerer.Accept()
		if err != nil {
			fmt.Println("监听客户端连接失败:", err)
			return
		}
		log.Println(conn.RemoteAddr(), "已经连接")
		go handlerconn(conn)
	}

}

func handlerconn(conn net.Conn) {
	defer conn.Close()
	// 接收客户端数据
	reader := bufio.NewReader(conn)
	for {
        // 解码
		str, err := proto.Decode(reader)
		if err != nil {
			fmt.Println("读取客户端数据失败:", err)
			return
		}
		log.Println("读取到的数据为:", str)
		// 发送数据给客户端
		msg := strings.ToUpper(str)
		// 编码
		data, err := proto.Encode(msg)
		if err != nil {
			log.Println("消息编码失败", err)
		}
		conn.Write(data)

	}

}
```

客户端代码
```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"proto"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:8080")
	if err != nil {
		log.Println("连接服务端失败:", err)
		return
	}
	defer conn.Close()
	go connServer(conn)
	reader := bufio.NewReader(conn)
	for {
        // 解码
		msg, err := proto.Decode(reader)
		if err != nil {
			fmt.Println("从服务器读取数据失败：", err)
			return
		}
		fmt.Println("从服务器读取的数据为：", msg)
	}
}

func connServer(conn net.Conn) {

	for i := 0; i < 20; i++ {
		msg := "how are you?"
        // 编码
		data, err := proto.Encode(msg)
		if err != nil {
			log.Println("消息编码失败", err)
		}
		// 给服务器发送数据
		_, err2 := conn.Write(data)
		if err2 != nil {
			fmt.Println("向服务器发送数据失败：", err2)
		}
	}
}
```