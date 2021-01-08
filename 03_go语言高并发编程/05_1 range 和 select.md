## range

通过 range 可以持续的从 channel 中读取数据,无论是有缓冲的 channel 还是无缓冲的 channel,都可以使用 range,当 channel 中没有数据的时候,会阻塞当前 goroutine,与读 channel 阻塞的机制相同

```go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int,3) // 创建无缓冲channel
    fmt.Println("len:",len(ch),"cap:",cap(ch))

    go func() {
        defer fmt.Println("子协程结束")
        for i :=0 ; i<5 ; i++ {
            ch <- i

        }
        close(ch)
    }()
    for data := range ch{
        fmt.Println("读取到的数据：",data)
    }
}
```

上面的代码中,在子 goroutine 中要使用 close()函数,明确的结束 channel,否则当检测到 range 遍历的 channel 所在的 goroutine 退出以后,会遇到 painc 异常



## select

go语言中提供了一个关键词 select,通过 select 可以监听 channel 上的数据流动

select 的使用与 switch 使用当时接近,使用 select 开始一个新的选择块,每个选择块中使用 case 语句来描述

与 switch 相比,select 语句有比较多的限制,其中最大的一条限制就是每个 case 语句里面必须是一个 io 操作,大致结构如下

```go
select {
    case <-chan1: // 此处必须是 IO 操作
        // 如果chan1成功读到数据，则进行该case处理语句
    case chan2 <- 1:
        // 如果成功向chan2写入数据，则进行该case处理语句
    default:
        // 如果上面都没有成功，则进入default处理流程
    }
```

在一个 select 语句中,go 语言会按照顺序从头到尾评估每一个接收和发送的语句,如果其中的任意一个语句可以执行(没有阻塞),那么就选择其中的任意一条来执行



如果没有语句可以执行,那么有两种情况

- 如果给出了 default 语句,那么就会执行 default 语句,同时程序的执行会从 select 语句后的语句中恢复
- 如果没有 default 语句,那么 select 语句会阻塞,直到有一个通道可以执行下去

> 一般在 select 语句中,不会去使用 default 语句,因为如果有了default,并且其他的语句都被阻塞的情况下,那么就会一直循环执行 select 语句,称为忙轮询,会造成比较大的 CPU 消耗,如果没有 default 语句,那么没有可执行的分支的时候,就会阻塞,就会释放 CPU 资源

```go
package main

import (
    "fmt"
    "runtime"
    "strings"
    "time"
)

func main() {
    ch := make(chan int)    // 用来传输数据的channel
    quit := make(chan bool) // 创建channel，用来终止程序

    go func() {
        for i :=0 ; i<10 ; i++ {
            if i == 5 {
                quit <- true
                runtime.Goexit()
            }
            ch <- i
            time.Sleep(time.Second)
        }
        close(ch)

    }()

    for {
        select {
        case num := <- ch: // 读取 数据传输 channel 中的数据
            fmt.Println(num)
        case <-quit: // 读取到退出的 channel 退出程序
            return
        }
        fmt.Println(strings.Repeat("-",10))
    }
}
```

#### 使用 select 实现斐波那契数列

```go
package main

import (
    "fmt"
    "runtime"
)

func fibonacci(ch <- chan int,quit <-chan bool)  {
    for {
        select {
        case num := <-ch:
            fmt.Print(num, " ")
        case <-quit:
            runtime.Goexit() // 等价于return
        }
    }
}

func main()  {
    ch := make(chan int)
    quit := make(chan bool)

    go fibonacci(ch,quit)
    x , y := 1 , 1 // 前两个值固定为1

    for i :=0 ; i<20 ; i++ {
        ch <- x
        x , y = y ,x+y
    }
    quit <- true
}
```

