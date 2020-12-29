# Map
前面学习了数组和切片,但是数组和切片有一个最大的缺点,就是当数据量特别大的时候,我们要获取到数组或者切片中的某个元素的值,需要将数组或切片遍历一遍

字典结构:由键(key)和value(值)组成,字典中的键是不能重复的,可以根据任意的键查询到指定的值

# Map的定义和使用
字典类型的定义使用关键词map:
- 变量名 map[key_type]value_type: key_type是键的类型,value_type是值的类型,键和值的类型可以是不同的类型

```go
func main() {
	var m1 map[int]string  // 定义了一个空的map
	fmt.Println(m1)
}
```

```go
func main() {
	m1 := make(map[int]string)// 使用map创建一个空map,也可以在后面指定长度
	m1[1] = "aaa"  // 赋值操作m1[key] = value
	m1[2] = "bbb"
	fmt.Println(m1) // 输出map[1:aaa 2:bbb]
}
```

> 注意:map是无序的,每次输出的键值对的顺序可能是不同的,map的长度是自动扩容的,使用len函数可以返回map的键值对的个数

## map定义.初始化并完成遍历
```go
func main() {
	m1 := map[int]string{1:"张三",2:"李四",3:"王五"}
	fmt.Println(m1[1]) // 指定要输出的key

	for k,v := range m1{ // 使用range可以返回map的key和value
		fmt.Printf("key的值为:%d,value的值为:%s\n",k,v)
	}
}
```

## map 判断指定的key是否存在
```go
func main() {
	m1 := map[int]string{1:"张三",2:"李四",3:"王五"}
	value , status := m1[4]
	if status == true {
		fmt.Println("m1[1]=",value)
	}else {
		fmt.Println("key不存在")
	}
}
```

> m1[4]有两个返回值,第一个返回值为对应key的value,第二个返回值为对应的key是否存在,存在为true,不存在为false,如果key不存在,那么value为空

## 删除map中的元素
删除元素的时候需要指定key的值,会根据key删除对应的value
```go
func main() {
	m1 := map[int]string{1:"张三",2:"李四",3:"王五"}
	delete(m1,1) // 指定删除的map和要删除的key
	fmt.Println(m1)
}
```
> delete在删除map中元素的时候,如果指定删除的key不存在也不会报错

## map 作为函数的参数使用
map 作为函数的参数使用的时候,使用的也是引用传递
```go
func test(m map[int]string)  {
	delete(m,2)
}

func main() {
	m1 := map[int]string{1: "张三", 2: "李四", 3: "王五"}
	test(m1)
	fmt.Println(m1)  // 输出 map[1:张三 3:王五]
}
```

> 注意:map在函数中添加数据是会影响主调函数中的值的,要区别于数组和切片

## 练习: 输入20个字符,统计每个字符出现的次数
```go
package main

import "fmt"

func test(arr map[byte]int)  {
	str :=make([]byte,20)
	for i :=0 ; i< len(str) ;i++ {
		fmt.Scanf("%c",&str[i])
	}

	for j :=0 ; j < len(str) ; j++{
		arr[str[j]]++
	}
	for key,value := range arr {
		fmt.Printf("字母%c有%d个\n",key,value)
	}
}

func main()  {
	arr :=make(map[byte]int)

	test(arr)
}
```

## 练习2:封装 wcFunc() 函数
接收一段英文字符串str。返回一个map，记录str中每个“词”出现次数的。
如："I love my work and I love my family too"
```go
package main

import (
	"fmt"
	"strings"
)

func weFunc(str string)  map[string]int {
	arr := strings.Fields(str)
	m1 := map[string]int{}
	for i :=0 ; i < len(arr) ; i++ {
		m1[arr[i]]++
	}
	return m1
}

func main() {

	str  := "I love my work and I love my family too"

	m1 := weFunc(str)

	for k,v := range m1 {
		fmt.Printf("单词%s有%d个\n",k,v)
	}
}
```
> strings.Fields()函数能将指定的字符串切割,并返回字符串数组