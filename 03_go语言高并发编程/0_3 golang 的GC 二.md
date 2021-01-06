## go 语言的 GC

https://www.cnblogs.com/zkweb/p/7880099.html

#### GC bitmap

GC 在标记的时候,需要知道哪些地方包含了指针,之前提到的 bitmap 区域涵盖了 arena 区域中的指针信息,除此之外,GC 还需要制动栈空间上有哪些地方包含了指针

因为栈空间不属于 arena 渔区,栈空间的指针信息记录在`函数信息`里面

另外,GC 在分配对象的时候,也需要根据对象的类型设置 bitmap区域,来源的指针信息将会在`类型信息`里面

总结起来 go 中有以下的 GC bitmap

- bitmap 区域: 涵盖了 arena 区域,使用 2bit 表示一个指针大小
- 函数信息: 涵盖了函数的栈空间,使用 1bit 表示一个指针大小的内存(位于 stackmap.bytedata)
- 类型信息: 在分配对象时候会复制到 bitmap 区域,使用 1bit 表示一个指针大小的内存(位于_type.gcdata)

