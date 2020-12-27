## 查找重复的行

对文件做拷贝,打印,搜索,排序或者统计类似的事情的时候,都有一个差不多的程序结构: 一个处理输入的循环,在每个元素上执行计算处理,在处理的同时或者最后产生输出

#### 使用的关键函数的说明

### bufio
bufio包实现了有缓冲的I/O。它包装一个io.Reader或io.Writer接口对象，创建另一个也实现了该接口，且同时还提供了缓冲和一些文本I/O的帮助函数的对象。

1. bufio.NewScanner
bufio 是一个带有缓存的标准库对于写操作来说,在被发送到 socket 或者硬盘之前,IO 缓冲区提供了一个临时存储区来存放数据,缓冲区存储的数据,达到一定的容量之后才会被释放出来,进行下一步的存储,这种方式大大的减少了写操作或者是最终的系统调用被触发的次数,而对于读操作来说,缓存 IO 意味着每次操作都能读取到更多的数据,即减少了系统调用的次数,又通过以块为单位读取硬盘数据来更高效的使用底层硬件

Scanner 是 bufio 的扫描器模块,它读取输入,并将其拆成行或者单词,
```go
func NewScanner(r io.Reader) *Scanner
```

NewScanner 创建并返回一个从 r 读取数据的 Scanner

##### 从标准输入中读取数据: 方式 1
```go
input := bufio.NewScanner(os.Stdin)
for input.Scan() {
    fmt.Println("输出",input.Text())
}
```
上面的代码中,使用 NewScanner 创建并返回一个从标准输入读取数据的 Scanner,每次调用 input.Scan(),即读取下一行,并移除末尾的换行符,读取到的内容可以调用input.Text() 得到,Scan 函数在读取到一行是返回 true,不再有输入时返回 false,使用组合键Ctrl +D 退出循环输入

##### 从标准输入中读取数据: 方式 2
```go
func main() {
	reader := bufio.NewReader(os.Stdin)
	fmt.Println("请输入数据:")
	str, _ := reader.ReadString('\n')
	fmt.Println("你输入的内容是:", str)

}
```

##### 从文件中读取数据

1. 使用 os.open() 方法打开文件并读取文件
```go
func main() {
	str := "/Users/weiying/go/test2.go"

	file, err := os.Open(str)
	if err != nil {
		log.Println("文件打开失败:", err)
	}
	input := bufio.NewScanner(file)
	for input.Scan() {
		fmt.Println("输出", input.Text())
	}
}

// 或者使用 bufio.NewReader,这种方式为指定换行符后按行读取
func main() {
	str := "/Users/weiying/go/test2.go"

	file, err := os.Open(str)
	if err != nil {
		log.Println("文件打开失败:", err)
	}
	defer file.Close()
	input := bufio.NewReader(file)
	for {
		line, err := input.ReadString('\n')
		if err == io.EOF {
			return
		}
		if err != nil {
			fmt.Println("读取文件失败")
			return
		}
		fmt.Print(line)
	}

}
```

2. 使用 ioutil.ReadFile() 方法读取文件
```go
func ReadFile(filename string) ([]byte, error)
```
参数是指定的文件,返回字节切片和错误信息,Readfile() 方法从指定的文件中读取数据并返回读取的数据以及错误信息,成功调用返回的 err 为 nil 而不是 EOF,因为本函数定义为读取整个文件
```go
func main() {
	str := "/Users/weiying/go/test2.go"

	data, err := ioutil.ReadFile(str)
	if err != nil {
		log.Println("读取文件失败:", err)
	}
	for _, value := range strings.Split(string(data), "\n") {
		fmt.Println("输出:", value)
	}
}
```


#### 练习 : 如果有命令行参数指定了文件，那么读取文件内容输出，如果没有从标准输入读取数据输出
```go
func main() {
	count := make(map[string]int)
	files := os.Args[1:]

	// 判断是否有命令行参数
	if len(files) ==0 {
		// 从标准输入读取数据
		countline(os.Stdin,count)
	}else {
		// 从文件中读取数据
		for _, file := range files {
			//打开文件
			f ,err := os.Open(file)
			if err != nil {
				fmt.Println("打开文件失败：",err)
				return
			}
			countline(f,count)
			defer f.Close()
		}
	}
	for k,v := range count{
		fmt.Println(k,"出现了",v,"次")
	}
}
// 通过 NewScanner 读取文件或者命令函参数
func countline1(r *os.File,count map[string]int)  {
	// 读取数据
	input := bufio.NewScanner(r)
	for input.Scan() {
		count[input.Text()]++
	}
}
// 通过NewReader 读取文件
func countline(r *os.File,count map[string]int)  {
	// 读取数据
	input := bufio.NewReader(r)
	for {
		// 指定文件换行符
		line ,err := input.ReadString('\n')
		if err == io.EOF {
			//fmt.Println("文件读取结束：",err)
			return
		}
		if err != nil {
			fmt.Println("文件读取失败：",err)
			return
		}
		count[line]++
	}
}
```

#### 向文件中写入数据

方法 1 : bufio.NewWriter 方法
```go
func main() {
	// 以读写的权限打开指定的文件
	filename, err := os.OpenFile("/Users/weiying/Desktop/b.txt", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)

	defer filename.Close() // 延迟调用，结束的时候一定要关闭文件

	if err != nil {
		fmt.Println("打开文件失败", err)
		return
	}

	write := bufio.NewWriter(filename)
	for i := 0; i < 5; i++ {
		// 现将数据写入缓存
		write.WriteString("hello")
	}
	// 将缓存中的数据写入文件
	write.Flush()
}
```

方法二: 使用 ioutil.WriteFile
```go
func main() {
	str := "hello"

	err := ioutil.WriteFile("./b.text", []byte(str), 0644)

	if err != nil {
		panic(err)
		return
	}
}
```

### 从标准输入中读取数据,输出每行数据重复的次数
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	// 创建map ,用来存储重复的次数
	count := make(map[string]int)
	// 创建从标准输入读取数据的Scanner 对象
	input := bufio.NewScanner(os.Stdin)
	// 循环从标准输入读取数据
	for input.Scan() {
		// 将读取到的数据,计数
		count[input.Text()]++
	}
	// 打印数据和次数
	for str, num := range count {
		fmt.Printf("%s\t%d\n次", str, num)
	}
}
```

### 从文件中读取数据,判断重复的行
1. 方法一
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

// 定义函数计算重复的行
func LineCounts(f *os.File,counts map[string]int) {
	// 创建从文件中读取数据的Scanner 对象
	input := bufio.NewScanner(f)
	// 循环在文件中读取每行数据
	for input.Scan() {
		counts[input.Text()]++
	}

}

func main() {
	// 从命令行中获取文件列表
	files := os.Args[1:]
	if len(files) == 0 {
		fmt.Println(os.Stderr, "使用方法为 commadn file1 file2 ...:")
		return
	}
	for _, file := range files {
		counts := make(map[string]int)
		f, err := os.Open(file)
		if err != nil {
			fmt.Println(os.Stderr, "打开文件失败")
			return
		}
		defer f.Close()
		// 调用函数计算文件中重复的行
		LineCounts(f,counts)
		fmt.Printf("文件:%s中每行出现的次数为:\n", file)
		for str, num := range counts {
			fmt.Printf("%s\t%d次\n", str, num)
		}
	}
}
```

2. 方法二
```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"strings"
)

func main() {
	// 从命令行中获取文件列表
	files := os.Args[1:]
	if len(files) == 0 {
		fmt.Println(os.Stderr, "使用方法为 commadn file1 file2 ...:")
		return
	}
	for _, file := range files {
		counts := make(map[string]int)
		data, err := ioutil.ReadFile(file)
		if err != nil {
			fmt.Println(os.Stderr, "读取文件失败", err)
			return
		}
		for _, value := range strings.Split(string(data), "\n") {
			counts[value]++
		}
		fmt.Printf("文件:%s中每行出现的次数为:\n", file)
		for str, num := range counts {
			fmt.Printf("%s\t%d次\n", str, num)
		}
	}
}
```