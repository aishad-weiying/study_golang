## sort 包

go 语言的sort包实现了内置和用户自定义的排序



### 内置排序

排序是内部排序,因此它会更改指定的切片,也是就在源切片上做更改,而不是返回一个新的切片

1. 数字类型的排序

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	arr := []int {4,5,6,2,1,3}
	// 检查指定切片是否已经为排序之后的递增顺序
	ok := sort.IntsAreSorted(arr)
	fmt.Println(ok)
	if !ok {
        // 执行排序
		sort.Ints(arr)
		fmt.Println(arr)
	}
}

```

2. 字符串类型的排序

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	arr := []string {"b","a","d","c"}
	// 检查指定切片是否已经为排序之后的递增顺序
	ok := sort.StringsAreSorted(arr)
	if !ok {
		sort.Strings(arr)
		fmt.Println(arr)
	}
}
```

### 自定义排序

有时候我们希望通过除了自然排序以外的其它方式对集合进行排序,例如,我们想按照字符串的长度进行排序,而不是对字符串的首字母进行排序



使用自定义排序的的时候,我们需要使用`sort.Sort()`方法,

```go
func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}
```

该方法的参数为interface 类型的接口,要想实现该接口必须实现该接口中定义的方法

```go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

该接口中定义了三个方法,分别是

- Len() : 返回集合参数的个数
- Less() : 判断两个字符串的长度,返回的是bool类型
- Swap(): 将两个自字符串进行交换

该方法调用data.Len() 确定集合长度,调用Less和Swap 函数进行判断并替换

下面来实现自定义根据字符串的长度进行排序

```go
package main

import (
	"fmt"
	"sort"
)
// 创建内置 []string 类型的别名,因为内置类型不能实现方法
type SortArr []string
// 实现Len 方法
func (s SortArr) Len() int {
	return len(s)
}
// 实现Less 方法
func (s SortArr) Less(i, j int) bool {
	return len(s[i]) < len(s[j])
}
// 实现Swap方法
func (s SortArr) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}


func main() {
	arr := SortArr{"aa","b","cccc","ddd"}
	// 排序
	sort.Sort(arr)
	fmt.Println(arr)
}
```

