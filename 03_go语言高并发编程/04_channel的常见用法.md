## 无缓冲channel

无缓冲channel,指的是接手数据前没有保存能力的channel

这种类型的channel要求发送数据的goroutine和接收数据的goroutine同时准备好才能完成数据的发送和接收操作,否则,通道会导致先执行发送或者读取数据的goroutine阻塞等待

这种通道进行发送和接收的交互行为本身就是同步的,其中任意一个操作永远无法独立于另一个操作单独存在

- 阻塞: 由于某种原因没有数据到达,当前协程处于等待的状态,直至满足条件,才能解除阻塞
- 同步: 在两个或者多个协程之间,能够保持数据的一致性的机制

![img](images/e384bae1cc39bd938d28a68e2da039dc.png)

1. 在第 1 步，两个 goroutine 都到达通道，但哪个都没有开始执行发送或者接收。
2. 在第 2 步，左侧的 goroutine 将它的手伸进了通道，这模拟了向通道发送数据的行为。这时，这个 goroutine 会在通道中被锁住，直到交换完成。
3. 在第 3 步，右侧的 goroutine 将它的手放入通道，这模拟了从通道里接收数据。这个 goroutine 一样也会在通道中被锁住，直到交换完成。
4. 在第 4 步和第 5 步，进行交换，并最终，在第 6 步，两个 goroutine 都将它们的手从通道里拿出来，这模拟了被锁住的 goroutine 得到释放。两个 goroutine 现在都可以去做别的事情了。

> channel 至少要被应用于两个go程及以上

### 无缓冲channel的创建

如果没有指定缓冲区的容量,那么该通道就是同步的,因此会阻塞到发送者准备好发送数据和接收者准备好接收数据

```go
package main

import "fmt"

func main() {
    ch := make(chan int,0) // 创建无缓冲channel
    fmt.Println("len:",len(ch),"cap:",cap(ch)) 
    // 因为创建的是无缓冲通道,那么 len() 和 cap() 统计的数量都为 0

    go func() {
        for i :=0 ; i<3 ; i++ {
            ch <- i
            fmt.Println("子go程正在运行：",i)
        }
    }()

    for i:=0 ; i<3 ; i++ {
        num := <- ch
        fmt.Println("主go程正在工作",num)
    }
}
```

## 有缓冲的channel

有缓冲的channel是一种再被接收前能存储一个或者多个数据值的通道

这种类型的channel并不强制的要求goroutine之间必须同时完成数据的发送和接收,通道会阻塞发送和接收的条件也不相同

只有通道中没有要接收的数据的时候,接收的动作才会被阻塞

只有通道总没有缓冲区可以容纳要发送的数据的时候,发送的动作才会被阻塞

这就导致了缓冲通道和无缓冲通道之间有一个很大的不同,无缓冲通道保证进行发送和接收的goroutine会在同一时间进行数据交换,有缓冲的通道则没有这种保证

![img](images/1c1f032a2401bb0145f76dcb137d43db.png)

1. 在第 1 步，右侧的 goroutine 正在从通道接收一个值。

   

2. 在第 2 步，右侧的这个 goroutine独立完成了接收值的动作，而左侧的 goroutine 正在发送一个新值到通道里。

3. 在第 3 步，左侧的goroutine 还在向通道发送新值，而右侧的 goroutine 正在从通道接收另外一个值。这个步骤里的两个操作既不是同步的，也不会互相阻塞。

4. 最后，在第 4 步，所有的发送和接收都完成，而通道里还有几个值，也有一些空间可以存更多的值

### 有缓冲通道的创建

如果给定了一个缓冲区的容量,那么通道就是异步的,只要缓冲区有尚未使用的空间用于发送数据或者还包含可以接收的数据,那么通道就会无阻塞的运行

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int,3) // 创建无缓冲channel
    fmt.Println("len:",len(ch),"cap:",cap(ch))

    go func() {
        defer fmt.Println("子协程结束")
        for i :=0 ; i<5 ; i++ {

            fmt.Printf("子go程正在运行：%d,len:%d,cap:%d\n",i,len(ch),cap(ch))
            ch <- i
        }
    }()
    time.Sleep(2 * time.Second)
    for i:=0 ; i<5 ; i++ {
        num := <- ch
        fmt.Println("主go程正在工作",num)
    }
}
```

上面代码的输入如下

```go
len: 0 cap: 3
子go程正在运行：0,len:0,cap:3
子go程正在运行：1,len:1,cap:3
子go程正在运行：2,len:2,cap:3
子go程正在运行：3,len:3,cap:3
主go程正在工作 0
主go程正在工作 1
主go程正在工作 2
主go程正在工作 3
子go程正在运行：4,len:3,cap:3
子协程结束
主go程正在工作 4
```

> 理论上来说,当 channel 的容量为 3 的时候,那么当子 go 程向其中写入了 0 、1、2的时候应该就阻塞的进程,等待主 go 程读取数据后,channel中等待读取的数据小于3的时候,才能继续写入,那么为什么输出的时候,显示子go 程写入了 4 个数据呢?,或者是出现子 go 程还没写入主 go 程就读取到的情况呢?

实际上系统在工作的时候正如我们所想,就是子 go 程写入了三个数据后就阻塞了,等待主 go 程的读取,显示为写入 4 个数据的原因就是因为其中有 IO 延迟

当进程需要打印输出的时候,需要使用到硬件设备(显示器),因为IO 操作比较耗时,就是当主 go 程读取到了数据,下面要打印数据的时候,还没等打印呢,结果分配到的 CPU 时间到了,只能是交出当前的 CPU ,只能是当下次获得 CPU 以后才能继续处理,这就导致了我们的程序在输出上出现与预想中的差异

## 关闭channel

如果发送者知道没有跟多的数据需要发送到channel中的话,那么接收者也能及时的知道没有多余的数据可以接收,这样的话,将是有必要的,因为接收者可以停止不必要的接收等待,这就可以通过内置的close()函数来关闭channel实现

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int,3) // 创建无缓冲channel
    fmt.Println("len:",len(ch),"cap:",cap(ch))

    go func() {
        defer fmt.Println("子协程结束")
        for i :=0 ; i<5 ; i++ {

            fmt.Printf("子go程正在运行：%d,len:%d,cap:%d\n",i,len(ch),cap(ch))
            ch <- i

        }
        close(ch)
    }()
    time.Sleep(2 * time.Second)
    for {
        if num, ok := <-ch; ok == true {
            fmt.Println("主go程正在工作", num)
        } else {
            break
        }
        fmt.Println("完成")
    }
}
```

使用channel 的时候需要注意一下几点

1. channel不想文件那样需要经常去关闭,如果当你确定确实是没有要发送的数据了,或者是想结束range循环之类的,才会去关闭channel
2. 关闭channel后,无法像channel中写入数据(引发panic异常后导致接收立即返回零值)
3. 关闭channel后,可以继续从channel中读取数据
4. 对于 nil channel ,无论是接收还是发送,都会被阻塞
5. ok的值为true表示成功的从channel中接收到了数据,为false表示channel已经被关闭,而且其中已经没有值可以接收了