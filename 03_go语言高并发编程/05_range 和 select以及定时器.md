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



## 定时器

time.Timer 是一个定时器,代表未来的一个单一事件,你可以告诉 timer 需要等待多长时间

```go
type Timer struct {
    C <-chan Time
    r runtimeTimer
}
```

它提供一个单向读 channel,在定时的时间到达之前,没有写入数据 timer.C 就会一直阻塞,知道定时时间达到,操作系统会向 time.C 写入系统的当前时间

```go
package main

import (
    "fmt"
    "time"
)

func main()  {
    fmt.Println("系统的当前时间：",time.Now())

    // 创建定时器，并指定时长
    timer1 := time.NewTimer(time.Second * 2)
    now := <- timer1.C
    fmt.Println("执行后的时间：",now)
}
```

### 定时方法的实现

1. time.Sleep

```go
time.Sleep(2 * time.Second)
```

1. time.Timer

```go
// 创建定时器，并指定时长
    timer1 := time.NewTimer(time.Second * 2)
    now := <- timer1.C
    fmt.Println("执行后的时间：",now)
```

1. time.After
   在使用 time.NewTimer 函数的时候,需要先创建一个定时器,然后在从定时器中获取C <-chan Time,但是 after 相当于将这两步合并成一步了

```go
    chaner := <- time.After(2 * time.Second)
    fmt.Println("执行后的时间：",chaner)
```

### 定时器的关闭和重置

1. 关闭定时器

```go
package main

import (
    "fmt"
    "time"
)

func main()  {
    timer := time.NewTimer(5 * time.Second)
    go func() {
        <- timer.C
        fmt.Println("子go程结束，定时器时间到")
    }()
    timer.Stop()  // 停止定时器
    for {
        ;
    }
}
```

1. 定时器重置

```go
package main

import (
    "fmt"
    "time"
)

func main()  {
    timer := time.NewTimer(5 * time.Second)
    ok := timer.Reset(2 * time.Second)  //  返回的是重置是否成功的结果
    go func() {
        fmt.Println(ok)
        <- timer.C
        fmt.Println("子go程结束，定时器时间到")
    }()

    for {
        ;
    }
}
```

## 周期性定时器

time.Ticker 是一个周期性触发定时的定时器,它会按照一个时间间隔往 channel 发送当前系统时间,而 channel 的接收者可以以固定的时间间隔从 channel 中读取事件

```go
type Ticker struct {
    C <-chan Time
    r runtimeTimer
}
```

测试代码

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    ticker := time.NewTicker(time.Second) // 创建一个周期定时器
    ch := make(chan bool) // 创建channel，用来终止程序

    i := 0
    go func() {
        for { // 通过循环触发周期定时器
            newtime := <- ticker.C
            i++
            fmt.Println(newtime)
            if i == 5 {
                ch <- true  // 当i=5 的时候，会向ch 中写入true，被读取之前都是阻塞的
                //runtime.Goexit() // 退出子go程
            }
        }
    }()
    <- ch // 读取ch中的数据，读取出来后主GO程结束
}
```

