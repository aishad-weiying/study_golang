## flag 包
指针是实现标准库中 flag 包的关键技术,它使用命令行参数来设置对应变量的值,而这些命令行标志参数的变量可能会零散的分布在这个程序中

> 为了说明这一点,在一些早期的 echo 版本总,就包含了两个可选的命令行参数: -你用于指定忽略行位的换行符 , -s seq 用于指定分隔符,默认为空格

 下面为ehco4 的源码
https://github.com/adonovan/gopl.io/blob/master/ch2/echo4/main.go

```go
// Echo4 prints its command-line arguments.
package main

import (
	"flag"
	"fmt"
	"strings"
)

// 注册 flag,将解析的结果保存在对应类型的指针中
// 参数 1: 命令行标志参数的名字
// 参数 2: 该标志参数的默认值
// 参数 3: 该标志参数的描述信息
var n = flag.Bool("n", false, "omit trailing newline")

var sep = flag.String("s", " ", "separator")

func main() {
    // 解析命令行参数写入到注册的 flag 中
  // 如果解析命令行参数的时候,遇到错误,默认将打印相关的提示进行,然后调用 os.Exit(2) 终止程序
	flag.Parse()
    // 通过 flag.Args 访问非标志参数的普通命令行参数
	fmt.Print(strings.Join(flag.Args(), *sep))
	if !*n {
		fmt.Println()
	}
}
```

如果用户在命令行输入了无效的标志参数或者-h,--help,那么将答应打印所有的标志参数的名字,默认值和藐视信息等

```go
func Parse() {
	CommandLine.Parse(os.Args[1:])
}
```


### flag 参数

flag包支持的命令行参数类型有`bool`、`int`、`int64`、`uint`、`uint64`、`float` `float64`、`string`、`duration`。

| flag 参数      | 有效值                                                       |
| -------------- | ------------------------------------------------------------ |
| 字符串         | 合法字符串                                                   |
| 整数 flag      | 1234、0664、0x1234等类型，也可以是负数。                     |
| 浮点数 flag    | 合法浮点数                                                   |
| bool 类型 flag | 1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False。    |
| 时间段 flag    | 任何合法的时间段字符串。如”300ms”、”-1.5h”、”2h45m”<br/> 合法的单位有”ns”、”us” /“µs”、”ms”、”s”、”m”、”h”。 |


### flag.TypeVar()

上面的代码中,我们通过`flag.Type`或得的标志位参数的命令都是指针类型,那么如果想要使用现有的变量来设置标志位可以使用`flag.TypeVar`函数
```go
// 基本格式:flag.TypeVar(Type指针, flag名, 默认值, 帮助信息)  
var name string  
var age int  
var married bool  
var delay time.Duration  
flag.StringVar(&name, "name", "张三", "姓名")  
flag.IntVar(&age, "age", 18, "年龄")  
flag.BoolVar(&married, "married", false, "婚否")  
flag.DurationVar(&delay, "d", 0, "时间间隔")
```

### flag.Parse()

通过以上两种方式定义的命令的 flag 参数之后,需要通过调用`flag.Parse()`来对命令行参数进行解析,支持的命令行格式由一下几种

*   `-flag xxx` （使用空格，一个`-`符号）
    
*   `--flag xxx` （使用空格，两个`-`符号）
    
*   `-flag=xxx` （使用等号，一个`-`符号）
    
*   `--flag=xxx` （使用等号，两个`-`符号）
    

其中,bool 类型的参数必须使用等号的方式指定

> Flag解析在第一个非flag参数（单个”-“不是flag参数）之前停止，或者在终止符”–“之后停止。

## flag其他函数
```go
flag.Args()  ////返回命令行参数后的其他参数，以[]string类型
flag.NArg()  //返回命令行参数后的其他参数个数
flag.NFlag() //返回使用的命令行参数个数
```