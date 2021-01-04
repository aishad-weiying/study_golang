## select 的实现原理

golang 实现select 时,定义了一个数据结构表示每个 case 语句(含 default 实际上是一种特殊的 case),select 执行过程可以类比为一个函数,函数输入 case 数组,输出选中的 case,然后程序流程转到选中的 case 块



### select 的数据结构

源码包`src/runtime/select.go:acase`定义了保湿 case 语句的结构体

```go
type scase struct {
	c           *hchan         // chan
	elem        unsafe.Pointer // data element
	kind        uint16
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}
```

1. scase.c 为当前 case 语句所操作的 channel 指针,这也说明了一个 case 语句只能操作一个 channel
2. scase.kind表示该 case 的了类型,分别为读 channel,写 channel 和 default,三种类型分别有常量定义:

- caseRecv:case 语句中尝试读取 scase.c 中的数据
- caseSend: case 语句中尝试向 scase.c 中写入数据
- caseDefault: default 语句

3. scase.elem 表示缓冲区地址,根据 scase.kind 不同,有不同的用途

- scase.kind == caseRecv : scase.elem 表示读取出 channel 的数据存放地址
- scase.kind == caseSenf : scase.elem 表示将要写入 channel 的数据存放地址

### select 实现逻辑

源码包`src/runtime/select.go:selectgo()`定义了 select 选择 case 的函数

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {...}
```

函数参数:

- cas0 为 scase 数组的首地址,selectgo()就是从这些 scase 中找出一个返回
- order0 为一个两倍 cas0 数组长度的 buffer,保存 scase 随机序列 pollorder 和 scase 中 channel 地址序列 lockorder
  - pollorder : 每次 selectgo 执行都会把 scase 序列打乱,以达到随机检测case 的目的
  - lockorder : 所有 case 语句中 channel 序列,以达到去重,防止对 channel 加锁是重复加锁的目的
- ncases : 表示 scase 数组的长度

返回值:

- int: 选中的 case 编号,这个 case 编号跟代码一致
- bool: 是否成功从 channel 中读取了数据,如果选中的 case 是从 channel 中读取数据,则返回值表示是否读取成功

#### selectgo 实现的伪代码

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
  // 1. 锁定 scase 语句中所有的 channel
  // 2. 按照随机顺序检查 scase 中的 channel 是否 ready
  // 		2.1 如果 case 可读,则读取 channel 中的数据,解锁所有的 channel,然后返回(case index,true)
  // 		2.2 如果 case 可写,则将数据写入到 channel,解锁所有的 channel,然后返回(case index,false)
  // 		2.3 所有的 case 都未 ready,则解锁所有的 channel,然后返回(default index,false)
  // 3. 所有 case 都未 ready,且没有 default 语句
  // 		3.1 将当前协程加入到所有 channel 的等待队列
  // 		3.2 当将协程转入阻塞,等待被唤醒
  // 4. 唤醒后返回 channel 对应的 case index
  // 		4.1 如果是读操作,解锁所有的 channel,返回(case index,true)
  // 		4.2 如果是写操作,解锁所有的 channel,返回(case index,false)
}
```

> 对于读 channel 的 case 来说,如`case elem,ok := <- chan1`,如果 channel 有可能被其他协程关闭的情况下,一定要检测是否读取成功,因为 close 的 channel 也是有返回值的,此时 ok == false

## 总结

1. select 语句中除了 default 之外,每个 case 操作一个 channel,要么是独有,要么是写
2. select 语句中除了 default 之外,各个 case 的执行顺序是随机的
3. select 语句中如果没有 default 语句,则会阻塞等待 case
4. select 语句中读操作要判断是否读取成功,关闭的 channel 也是可以读取的

