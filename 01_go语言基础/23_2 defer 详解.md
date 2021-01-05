## defer 的规则

golang 官方总结了 defer 的规则,只有三条,下面我们来看一下这三条规则



### 1. 延迟函数的参数在 defer 语句出现之前就已经确定下来了

```go
func a() {
  i := 0
  defer fmt.Println(i)
  i++
  return
}
```

defer 语句中的`fmt.Println(i)`参数 i 的值,在 defer 出现的时候就已经确定下来了,实际上就是拷贝了一份,后面对变量 i 进行修改并不会影响`fmt.Println(i)`函数的执行,仍然打印'0'

>  注意: 对于指针来兴的参数,这个规则仍然适用,只不过是延迟调用的参数是一个地址值,这种情况下,defer 后面的语句对变量的修改可能会影响延迟调用

### 2. 延迟函数执行按照后进先出的顺序执行,即先出现的 defer 最后执行

这个规则很好理解,定义 defer 类似于入栈操作,执行 defer 类似出栈操作,比如先申请 A 资源,再根据 A 资源申请 B 资源,根据 B 资源申请 C 资源,即申请顺序是:A->B->C,释放的顺序与申请的顺序相反

> 每申请到一个用完需要释放的资源的时候,立即定义一个 defer 来释放资源是一个很好的习惯

### 3. 延迟函数可能操作主函数的具名返回值

定义 defer 的函数,即主函数可能有返回值,返回值有没有名字无所谓,defer 所作用的函数,即延迟函数可能会影响到返回值

要是理解延迟函数是如何影响主函数的返回值的,那么就要明白函数是如何返回的就足够了

#### 函数返回过程

有一个实时必须了解,关键字 return 并不是一个原子操作,实际上 return 只代理汇编指令 ret,即将跳转程序执行,比如`return i`,实际上分为两步操作,即将 i 的值入栈中作为返回值,然后执行跳转,而 defer 的执行时机正是跳转之前,所以说 defer 执行时还是有机会操作返回值的

```go
func deferFuncReturn()(result int){
  i :=1
  defer func(){
    result ++
  }()
  return i
}
```

该函数的 return 执行可以分为以下两步

```go
result = i
return
```

而 defer 函数的执行正是在 return 执行之前

```go
result = i
result++
return
```

#### 主函数拥有匿名返回值,返回字面值

一个主函数拥有一个匿名返回值,返回时使用字面值,比如返回`1,2,hello`这样的字面值的时候,defer 不能操作返回值

```go
func foo()int {
  i := 0
  defer func(){
    i++
  }()
  return 1
}
```

上面的语句在执行 return 的时候,直接把 1 写入栈中作为返回值,延迟函数无法操作该返回值,所以无法影响返回值

#### 主函数用用匿名返回值,返回变量

一个主函数用于一个匿名返回值,返回使用本地或者全局变量,这种情况下 defer 语句可以引用到返回值,但是不会更改返回值变量

```go
func foo() int {
  var i int 
  defer func(){
    i++
  }()
  return i
}
```

 上面的函数,返回一个局部变量,同时 defer 函数也会操作这个局部变量,对于匿名返回值来说,可以假定仍然有一个变量存储返回值,假如有一个变量`a`存储返回值,那么上面的语句仍然可以拆分为

```go
a = i
i++
return
```

由于 i 是整型,回家值拷贝给 a 变量,所以 defer 语句中修改 i 的值,对函数的返回值不会造成影响

#### 主函数拥有具名返回值

主函数语句中带名字的返回值,会被初始化为一个局部变量,函数内部可以像使用局部变量一样使用该返回值,如果 defer 语句操作该返回值,可能会改变返回结果

```go
func deferFuncReturn()(result int){
  i :=1
  defer func(){
    result ++
  }()
  return i
}
```

### defer 实现原理

源码包`src/src/runtime/runtime2.go:_defer`定义了 defer 的数据结构

```go
type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	// openDefer indicates that this _defer is for a frame with open-coded
	// defers. We have only one defer record for the entire frame (which may
	// currently have 0, 1, or more defers active).
	openDefer bool
	sp        uintptr  // sp at time of defer 函数栈指针
	pc        uintptr  // pc at time of defer 程序计数器
	fn        *funcval // can be nil for open-coded defers  函数地址
	_panic    *_panic  // panic that is running defer
	link      *_defer  // 指向自身结构的指针,用于链接多个 defer

	// If openDefer is true, the fields below record values about the stack
	// frame and associated function that has the open-coded defer(s). sp
	// above will be the sp for the frame, and pc will be address of the
	// deferreturn call in the function.
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr
}
```

 我们知道 defer 后面一定要接一个函数,所以 defer 的数据结构跟一般的函数类似,也有栈地址,程序计数器,函数地址等

与函数不同的一点是,还含有一个指针,可用于指向另一个 defer,每个 goroutine 数据结构中实际也有一个 defer 指针,该指针指向一个 defer 的单链表,每次声明一个 defer 时就将 defer 插入到单链表的表头,每次执行 defer 时就从表头中取出一个 defer 执行

> 新声明的 defer 总是添加到链表头部

函数返回前执行 defer,就是从链表头部依次取出执行,进入函数的时候添加 defer,离开函数的时候取出 defer,所以,即便是调用多个函数,也能保证 defer 是按照后进先出(FIFO)的方式执行的



### defer 的创建和执行

源码包`src/runtime/panic.go`中定义了两个方法分别用于创建 defer 和执行 defer

- deferproc(): 在声明 defer 处调用,将其 defer 函数存入 goroutine 的链表中
- deferreturn(): 在 return 执行,确切的讲是 ret 指令调用之前,将其 defer 从 goroutine 链表总取出执行

可以简单的理解为,在编译阶段,声明 defer 处插入了函数deferproc(),在函数 return 之前插入函数deferreturn()

## 总结

1. defer 定义的延迟函数菜蔬在 defer 语句出现的时候就已经确定下来了
2. defer 定义顺序与实际执行顺序相反
3. return 不是原子操作,执行过程是: 保存返回值->执行 defer->执行 ret 跳转
4. 申请资源后要立即使用 defer 关闭资源



