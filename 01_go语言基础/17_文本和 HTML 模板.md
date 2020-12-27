## 文本和 HTML 模板

有的时候我们需要打印一些复杂的格式,这时一般需要将格式化代码分离出来以便安全的修改,这些功能由 text/template 和 html/template 等模板包提供,它们提供了将一个变量填充到一个文本或者 HTML 格式的模板的机制

一个模板是一个字符串或一个文件,里面包含了一个或多个由双花括号包含的{{action}}对象,大部分的字符串只是按照字面值进行打印,但是对于 actions 部分将处罚其他的行为,每个 actions 都包含了一个用模板语言编写的表达式,一个 action 虽然简短,但是可以输出复杂的打印值

```go
const templ = `{{.TotalCount}} issues:
{{range .Items}}----------------------------------------
Number: {{.Number}}
User:   {{.User.Login}}
Title:  {{.Title | printf "%.64s"}}
Age:    {{.CreatedAt | daysAgo}} days
{{end}}`
```

这个首先打印匹配到的 issues 的总数,然后打印每个issue的编号、创建用户、标题还有存在的时间。

都有一个当前值的概念，对应点操作符，写作“.”。当前值“.”最初被初始化为调用模板时的参数，在当前例子中对应github.IssuesSearchResult类型的变量。模板中{{.TotalCount}}对应action将展开为结构体中TotalCount成员以默认的方式打印的值。模板中{{range .Items}}和{{end}}对应一个循环action，因此它们直接的内容可能会被展开多次，循环每次迭代的当前值对应当前的Items元素的值。

在各个 action 中, **| 操作符** 表示将前一个表达式的结果作为后一个函数的输入,类型 UNIX 系统中管道的概念,在Title这一行的action中，第二个操作是一个printf函数，是一个基于fmt.Sprintf实现的内置函数，所有模板都可以直接使用。对于Age部分，第二个动作是一个叫daysAgo的函数，通过time.Since函数将CreatedAt成员转换为过去的时间长度：
```go
func daysAgo(t time.Time) int {
    return int(time.Since(t).Hours() / 24)
}
```

生成模板一般需要两个步骤,第一步是要分析模板并转为内部表示,然后基于指定的输入执行模板

而分析模板只需要执行一次,分析模板的步骤

- 调用 template.New 先创建并返回一个模板
- Funcs 方法将daysAgo等自定义函数注册到模板中,并返回模板
- 最后调用 Parse 函数分析模板
```go
report, err := template.New("report").
    Funcs(template.FuncMap{"daysAgo": daysAgo}).
    Parse(templ)
if err != nil {
    log.Fatal(err)
}
```

因为模板通常在编译时就测试好了,如果模板解析失败将是一个致命的错误,template.Must辅助函数可以简化这个致命错误的处理：它接受一个模板和一个error类型的参数，检测error是否为nil（如果不是nil则发出panic异常），然后返回传入的模板。
```go
var report = template.Must(template.New("issuelist").
    Funcs(template.FuncMap{"daysAgo": daysAgo}).
    Parse(templ))
```