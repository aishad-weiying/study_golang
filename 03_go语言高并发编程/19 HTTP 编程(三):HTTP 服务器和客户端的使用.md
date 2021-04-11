Go 语言标准库中内建提供了 net/http 包,涵盖了 http 客户端和服务器端的具体实现,使用 net/http 包,我们可以很轻便的编写 http 客户端和服务器端的程序

#### 基本的 http 请求
```go
resp, err := http.Get("http://example.com/")
...
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
...
// 发送表单
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})
```

#### 简单的服务器端的代码
```go
package main

import (
	"fmt"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request)  {
	// w 表示写回给客户端的数据 ， r表示从客户端读取的数据
	fmt.Println("method=",r.Method)  // 请求方法
	fmt.Println("URL=",r.URL)		// 请求路径
	fmt.Println("header=",r.Header) // 请求头
	fmt.Println("body=",r.Body)	//请求包体
	fmt.Println(r.RemoteAddr,"连接成功") // 获取客户端地址

	w.Write([]byte("hello"))	// 发送数据给客户端
    // 也可以使用 Fprintf 方法直接将数据写入到 w 中
    fmt.Fprintf(w, "Host = %q\n", r.Host)
}

func main()  {
	// 注册回调函数
	http.HandleFunc("/index.html",handler)

	// 绑定服务器监听的地址
	http.ListenAndServe("127.0.0.1:8080",nil)
}
```
http.HandleFunc() 函数有两个参数,第一个为请求的路径,第二个参数为要调用的回调函数,当触发第一个参数的时候会自动的调用回调函数

回调函数的参数固定有两个,分别为http.ResponseWriter 和 *http.Request, 分别是用来给客户端发送数据和从客户端接收数据的
```go
type ResponseWriter interface {
	Header() Header
	Write([]byte) (int, error)
	WriteHeader(statusCode int)
}

type Request struct {
	Method string		// 浏览器请求方法 GET、POST…
	URL *url.URL		// 浏览器请求的访问路径
	……
	Header Header		// 请求头部
	Body io.ReadCloser	// 请求包体
	RemoteAddr string	// 浏览器地址
	……
	ctx context.Context
}
```

http.ListenAndServe 函数用于在指定的 tcp 网络地址进行监听,然后调用服务端处理程序来处理传入的连接请求,该方法有两个参数,第一个参数是监听的地址和端口,第二个参数表示服务器端处理程序,通常为 nil,当为 nil 的时候,服务器端会自动的调用自带的回调函数 http.DefaultServeMux 进行处理

### 练习
在计算机中选定一个目录,存放一些文件,当用户访问的时候,可以根据访问路径访问下面的文件
```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func sendFile(file string,w http.ResponseWriter)  {
	//打开指定的文件
	f , err := os.Open(file)
	if err != nil {
		fmt.Println("os.open error",err)
		w.Write([]byte("访问的文件不存在！！！"))
		return
	}
	defer f.Close()
	buf := make([]byte,4096)
	for {
		n ,err := f.Read(buf)
		if n == 0{
			return
		}
		if err != nil {
			fmt.Println("os.open error",err)
			return
		}
		w.Write(buf[:n])
	}
}

func handler(w http.ResponseWriter, r *http.Request)  {
	// w 表示写回给客户端的数据 ， r表示从客户端读取的数据
	file := "/Users/weiying/Desktop" + r.URL.String()
	sendFile(file,w)
}

func main()  {
	// 注册回调函数
	http.HandleFunc("/",handler)

	// 绑定服务器监听的地址
	http.ListenAndServe("127.0.0.1:8080",nil)

}
```
浏览器访问测试
![](images/94a02edbe8d687b5acd76d6565749683.png)

## HTTP客户端
客户端访问 web 服务器数据,之哟啊使用 http.get函数来完成,读取到的响应报文存储在 response 结构体中

```go
func Get(url string) (resp *Response, err error) {
	return DefaultClient.Get(url)
}
// 参数为 URL,必须包含 http
//返回响应报文和错误信息

// response 结构体
type Response struct {
	Status     string // e.g. "200 OK"
	StatusCode int    // e.g. 200
	Proto      string // e.g. "HTTP/1.0"
	ProtoMajor int    // e.g. 1
	ProtoMinor int    // e.g. 0
	Header Header  // 响应包头
	Body io.ReadCloser // 响应包体
	ContentLength int64
	TransferEncoding []string
	//Close在服务端指定是否在回复请求后关闭连接，在客户端指定是否在发送请求后关闭连接。
	Close bool
	Uncompressed bool
	Trailer Header
	Request *Request
	TLS *tls.ConnectionState
}
```
服务器发送的数据主要保存在 Body 包体中,可以使用 read 方法读取,保存在切片缓冲区中,结束的时候需要使用 close() 方法关闭 io

#### 客户端代码
```go
package main

import (
	"fmt"
	"net/http"
)

func main()  {
	// 使用Get方法获取服务器响应包数据
	resp ,err :=http.Get("http://www.baidu.com")
	if err != nil{
		fmt.Println("http.get error:",err)
		return
	}
	defer resp.Body.Close()
	fmt.Println("Status = ", resp.Status)           // 状态
	fmt.Println("StatusCode = ", resp.StatusCode)   // 状态码
	fmt.Println("Header = ", resp.Header)           // 响应头部
	//fmt.Println("Body = ", resp.Body)               // 响应包体

	// 定义切片缓冲区，存读到的内容
	buf := make([]byte,4096)
	var result string
 	for {
		n , err := resp.Body.Read(buf)
		if n == 0 {
			break
		}
		if err != nil{
			fmt.Println("resp.Body.Read error:",err)
			break
		}
		result += string(buf[:n])
	}
	fmt.Println(result)
}
```

#### 特殊情况下,自定义发送请求
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"net/url"
)

func main() {
	// 首先创建请求的url
	data := url.Values{}
	urlObj, _ := url.Parse("http://127.0.0.1:8080/index")
	// 设置请求的参数
	data.Set("name", "张三")
	data.Set("age", "100")
	// 编码url参数,防止出现特殊字符影响url
	queryStr := data.Encode()
	fmt.Println("encode之后的url参数:", queryStr)
	// 合并url与参数
	urlObj.RawQuery = queryStr
	// 创建请求
	request, _ := http.NewRequest("GET", urlObj.String(), nil)
	//发送请求
	respone, _ := http.DefaultClient.Do(request)
	buf, _ := ioutil.ReadAll(respone.Body)
	fmt.Println(string(buf))
	defer respone.Body.Close()
}
```

服务器端接收客户端发送的参数
```go
func handleconn(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
	queryParam := r.URL.Query()
	fmt.Println(queryParam)
	fmt.Println(queryParam.Get("name"))
	fmt.Println(queryParam.Get("age"))
}
```

### 自定义发送 post 请求
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

// net/http post demo

func main() {
	url := "http://127.0.0.1:9090/post"
	// 表单数据
	//contentType := "application/x-www-form-urlencoded"
	//data := "name=小王子&age=18"
	// json
	contentType := "application/json"
	data := `{"name":"小王子","age":18}`
	resp, err := http.Post(url, contentType, strings.NewReader(data))
	if err != nil {
		fmt.Printf("post failed, err:%v\n", err)
		return
	}
	defer resp.Body.Close()
	b, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("get resp failed, err:%v\n", err)
		return
	}
	fmt.Println(string(b))
}
```

对应的 server 端 HandlerFunc 如下:
```go
func postHandler(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()
	// 1. 请求类型是application/x-www-form-urlencoded时解析form数据
	r.ParseForm()
	fmt.Println(r.PostForm) // 打印form数据
	fmt.Println(r.PostForm.Get("name"), r.PostForm.Get("age"))
	// 2. 请求类型是application/json时从r.Body读取数据
	b, err := ioutil.ReadAll(r.Body)
	if err != nil {
		fmt.Printf("read request.Body failed, err:%v\n", err)
		return
	}
	fmt.Println(string(b))
	answer := `{"status": "ok"}`
	w.Write([]byte(answer))
}
```


### http 建立短链接

```go
func main() {
	// 首先创建请求的url
	data := url.Values{}
	urlObj, _ := url.Parse("http://127.0.0.1:8080/index")
	// 设置请求的参数
	data.Set("name", "张三")
	data.Set("age", "100")
	// 编码url参数,防止出现特殊字符影响url
	queryStr := data.Encode()
	fmt.Println("encode之后的url参数:", queryStr)
	// 合并url与参数
	urlObj.RawQuery = queryStr
	// 创建请求
	tr := &http.Transport{
		DisableKeepAlives: true,
	}
	client := http.Client{
		Transport: tr,
	}
	request, _ := http.NewRequest("GET", urlObj.String(), nil)
	//发送请求
	respone ,_ := client.Do(request)
	buf, _ := ioutil.ReadAll(respone.Body)
	fmt.Println(string(buf))
	defer respone.Body.Close()
}
```

短链接使用的场景: 连接的请求不是特别频繁,请求完毕之后马上关闭连接,而不是持续的占有长连接

当要将连接复用的时候,例如,每 5 秒钟发送一下 http 请求,那么我们就可以定义全局的长连接 client,然后在代码中复用这个连接,如果请求不是很频繁的话,那么依旧是使用短链接,并且是局部变量,每次使用的时候,重新创建短链接去请求