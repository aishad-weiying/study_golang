## range 的实现原理

for-range 的语句的实现,在源码包`gofrontend/go/statements.cc/For_rang_statment:do_lower()`方法中有如下注释

```go
 // Arrange to do a loop appropriate for the type. We will produce
 // for INIT ; COND ; POST {
 // ITER_INIT
 // INDEX = INDEX_TEMP
 // VALUE = VALUE_TEMP // If there is a value
 // original statements
 // }

```

可见range实际上是一个C风格的循环结构,range支持数组,数组指针,切片,map和channel类型,对于不同类型有些细节上的差别

### range 遍历 slice

变量slice会先获取以slice的长度len_temp作为循环次数,循环体重,每次循环会先获取元素值,如果for-range中接收index和value的话,则会对index和value进行一次赋值

由于循环开始前循环的次数就已经确定了,所以循环过程中新添加的元素是不能遍历到的

另外,数组与数组指针的遍历过程中与slice基本一致

### range 遍历 map

遍历 map 时没有指定循环次数,循环体与遍历slice,由于底层实现与slice不同,map底层使用hash表实现,插入数据的位置是随机的,所以遍历过程中新插入的数据不能保证遍历到

### range 遍历 channel

range 遍历channel是最特殊的,这是由channel的实现机制决定的

```go
 // The loop we generate:
 // for {
 // index_temp, ok_temp = <-range
 // if !ok_temp {
 // break
 // }
 // index = index_temp
 // original body
 // }

```



channel遍历是依次从channel中读取数据,读取前是不知道里面有多少元素的,如果channel中没有元素,则会阻塞等待,如果channel已经被关闭,则会解除阻塞并退出循环

注:

- 上述注释中index_temp实际上描述是有误的,应该为value_temp,因此index对于channel是没有意义的
- 使用for-range变量channel时只能获取一个返回值

### 编程Tips

- 遍历过程汇总可以视情况放弃接收index或者value,可以一定程度上提升性能
- 遍历channel时,如果channel中没有数据,可能会阻塞

## 总结

- for-range的实现实际上是C风格的for循环
- 使用index,value接收range返回值会发生一次数据拷贝