## 获取 URL
为了简单的展示基于 HTTP 获取信息的方式,下面给出一个实例程序 fetch,这个程序将获取对应的 url,并将其源文件打印出来,类似 curl 工具

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
)

func main() {
	for _, url := range os.Args[1:] {
		// 如果不是以http:// 开头需要添加
		if strings.HasPrefix(url, "http://") == false {
			url = "http://" + url
		}
		res, err := http.Get(url)
		if err != nil {
			fmt.Println("http.get error:", err)
			return
		}
		defer res.Body.Close()
        b, _ := ioutil.ReadAll(res.Body)
		fmt.Println(string(b))
		// io.Copy(os.Stdout, res.Body)
	}
}

```

> 上面的代码中中可以使用 io.Cpoy 直接输出到标准输出,而不用通过缓冲区(b)来存储


## 并发获取多个 url

上面的代码中.如果要获取多个 URL 只能是根据循环一个个获取不能同时进行.这里可以使用 goroutine 和 channel 来实现并发
```go
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"os"
	"time"
)

func main() {
	// 创建channel ,让主go程在获取完毕所有的数据之后才退出
	ch := make(chan string)
	// 循环各个url
	for _, url := range os.Args[1:] {
		// 创建go程,获取url
		go HttpGET(url, ch)

		fmt.Println(<-ch)
	}
}
// 爬去http 数据
func HttpGET(url string, ch chan string) {
	start := time.Now()
	resp, err := http.Get(url)
	if err != nil {
		fmt.Println("请求 URL 错误:", err)
		ch <- ""
		return
	}
	defer resp.Body.Close()
	// ioutil.Discard 相当于 /dev/null
	nbytes, _ := io.Copy(ioutil.Discard, resp.Body)
	secs := time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}

```