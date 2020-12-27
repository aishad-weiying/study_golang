## Time 包的用法

time.Time 类型表示时间,我们可以通过 time.Now() 函数获取当前的时间对象,然后获取时间对象的年月日时分秒等信息

```go
// time.Now(), 返回值为一个时间对象
func Now() Time 

// 根据对象可以获取其他的时间信息
now := time.Now()
year := now.Year()
month := now.Month()
day := now.Day()
hour := now.Hour()
min := now.Minute()
second := now.Second()
```

#### 时间戳

时间戳是自 1970 年1 月 1 日到当前时间的总毫秒数,也被称为 unix 时间戳,基于时间戳获取对象如下

```go
fmt.Println(time.Now().Unix())        // 毫秒级别的时间戳
fmt.Println(time.Now().UnixNano())    // 纳秒时间戳
```

#### 使用 time.Unix() 函数可以将时间戳转换为时间格式
```go
// 参数为时间戳和nsec的值在[0, 999999999]范围外是合法的,返回值为 time 对象
now := time.Unix(1595208012, 0)
fmt.Println(now,now.Year())
```

#### 时间间隔
```go
const (
	Nanosecond  Duration = 1
	Microsecond          = 1000 * Nanosecond
	Millisecond          = 1000 * Microsecond
	Second               = 1000 * Millisecond
	Minute               = 60 * Second
	Hour                 = 60 * Minute
)
```

### 时间操作

1. Add
时间+时间间隔
```go
// func (t Time) Add(d Duration) Time
// 参数为要添加的时间间隔,返回值为 time 对象
fmt.Println(time.Now().Add(time.Hour))
```

2. Sub 计算两个时间的差值
```go
func (t Time) Sub(u Time) Duration
```

3. Equal 判断两个时间是否相同,不同的时区也可以比较
```go
func (t Time) Equal(u Time) bool
```

4. Before
判断 t 代表的时间是否在 u 代表的时间之前
```go
func (t Time) Before(u Time) bool
```

5. After 
判断 t 代表的时间是否在 u 代表的时间之后
```go
func (t Time) After(u Time) bool
```

### 使用go语言获取当前时间之前的指定一段的时间

1. 获取当前时间之前 n年或者n月或者n天的时间

```go
func (t Time) AddDate(years int, months int, days int) Time
```

AddDate 方法有三个参数,第一个参数为年,第二个参数为月,第三个参数为天,如果要获取当前时间的前一年的时间,方式如下

```go
time.Now().AddDate(-1,0,0)
```

2. 获取当期时间之前或者之后的指定时间间隔的时间

```go
// 获取当期时间一个小时之前的时间
time.Now().Add(-time.Hour)
// 获取当前时间一个小时之后的时间
time.Now().Add(time.Hour)
```



### 定时器
使用 time.Tick(时间间隔) 来设置定时器
```go
ticker := time.Tick(time.Second)
	for i := range ticker {
		fmt.Println(i)
	}
```

定时器本质上就是一个管道
```go
func Tick(d Duration) <-chan Time {
	if d <= 0 {
		return nil
	}
	return NewTicker(d).C
}
```

### 时间格式化
时间类型有一个自带的 Fromat 进行格式化,但是 Go 语言使用的时间是 2006-01-02 15:04:05,格式化就是将时间类型的对象,转换为字符串类型
```go
fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
fmt.Println(time.Now().Format("2006/01/02 15:04:05"))
fmt.Println(time.Now().Format("2006/01/02 03:04:05 PM"))
fmt.Println(time.Now().Format("2006/01/02 15:04:05.0000"))
```

#### 将字符串类型的时间转化为 time 对象
```go
func Parse(layout, value string) (Time, error)

// 参数为时间格式和字符串类型的时间
now, err := time.Parse("2006/01/02 15:04:05", "2020/07/20 10:03:55")
	if err != nil {
		return
	}
fmt.Println(now.Unix())
```

#### 根据指定的时区获取对象
```go
// func LoadLocation(name string) (*Location, error)

loc, err := time.LoadLocation("Asia/Shanghai")
	if err != nil {
		fmt.Println(err)
	}
// 将指定的字符串和时区获取时间对象
// func ParseInLocation(layout, value string, loc *Location) (Time, error)
now, err := time.ParseInLocation("2006-01-02 15:04:05", "2020-07-20 09:47:24", loc)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(now)
```


## log 包的用法
Go 语言内置的 `log` 包实现了简单的日志服务
log包定义了Logger类型，该类型提供了一些格式化输出的方法。本包也提供了一个预定义的“标准”logger，可以通过调用函数   `Print系列`(Print|Printf|Println）、`Fatal系列`（Fatal|Fatalf|Fatalln）、和`Panic系列`（Panic|Panicf|Panicln）来使用，比自行创建一个logger对象更容易使用。

```go
package main

import (
	"log"
)

func main() {
	log.Println("这是一条很普通的日志。")
	v := "很普通的"
	log.Printf("这是一条%s日志。n", v)
	log.Fatalln("这是一条会触发fatal的日志。")
	log.Panicln("这是一条会触发panic的日志。")
}
```

### 配置 logger
默认情况下的 logger 只会为我们提供日志的时间信息,但是很多情况下,我们希望得到更到的信息,比如记录该日志的文件名和行号等,`log`标准库为我们提供了定制这些设置的方法

`log`标准库中的`Flags`函数会返回标准 logger 的输出配置,而`SetFlags`函数用来设置标准 logger的输出配置

`log`标准库提供了如下的 flag 选项
```go
const (
    // 控制输出日志信息的细节，不能控制输出的顺序和格式。
    // 输出的日志在每一项后会有一个冒号分隔：例如2009/01/23 01:23:23.123123 /a/b/c/d.go:23: message
    Ldate         = 1 << iota     // 日期：2009/01/23
    Ltime                         // 时间：01:23:23
    Lmicroseconds                 // 微秒级别的时间：01:23:23.123123（用于增强Ltime位）
    Llongfile                     // 文件全路径名+行号： /a/b/c/d.go:23
    Lshortfile                    // 文件名+行号：d.go:23（会覆盖掉Llongfile）
    LUTC                          // 使用UTC时间
    LstdFlags     = Ldate | Ltime // 标准logger的初始值
)
```

设置日志输出结果
```go
func main() {
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
	log.Println("这是一条很普通的日志。")
}
```

#### 设置日志前缀
`log`标准库中还提供了关于日志信息前缀的两个方法

```go
func Prefix() string
func SetPrefix(prefix string)
```

其中 `Prefix` 函数用来查看标准 logger 的输出前缀,`SetPrefix` 函数用来设置输出前缀
```go
func main() {
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
	log.Println("这是一条很普通的日志。")
	log.SetPrefix("[小王子]")
	log.Println("这是一条很普通的日志。")
}
```

#### 配置日志输出的位置
`SetOutput` 函数用来设置标准 logger 的输出目的地,默认是标准错误输出
```go
func SetOutput(w io.Writer) {
	std.mu.Lock()
	defer std.mu.Unlock()
	std.out = w
}
```

把日志输出到文件中
```go
package main

import (
	"fmt"
	"log"
	"os"
)

func main() {
	logFile, err := os.OpenFile("./xx.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		fmt.Println("open log file failed, err:", err)
		return
	}
	log.SetOutput(logFile)
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
	log.Println("这是一条很普通的日志。")
	log.SetPrefix("[小王子]")
	log.Println("这是一条很普通的日志。")
}
```

一般在使用标准的 logger 的时候,会把上面的内容定义在 `init`函数中
```go
func init() {
	logFile, err := os.OpenFile("./xx.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		fmt.Println("open log file failed, err:", err)
		return
	}
	log.SetOutput(logFile)
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
}
```

### 创建 logger
`log` 标准库中还提供了一个创建新 logger 对象的构造函数`New`,支持自定义 logg 而示例
```go
func New(out io.Writer, prefix string, flag int) *Logger
```

New 创建一个 logger 对象,其中,参加数 out 设置日志信息写入的目的地,参数 prefix 会添加到生成的每条日志的前面,参数flag定义日志的属性（时间、文件等等）。

```go
func main() {
	logger := log.New(os.Stdout, "<New>", log.Lshortfile|log.Ldate|log.Ltime)
	logger.Println("这是自定义的logger记录的日志。")
}
```

Go内置的log库功能有限，例如无法满足记录不同级别日志的情况，我们在实际的项目中根据自己的需要选择使用第三方的日志库，如logrus、zap等。

## 自定义日志库

1. 日志要支持往不同的地方输出
2. 日志级别
- Debug
- Trace
- Info
- Warning
- Error
- Fatal

3. 日志要支持开关控制
4. 日志要有时间,行号,文件名,日志级别和日志信息
5. 日志要切割

#### 首先实现往终端中输出日志
```go
package mylogger

import (
	"log"
	"os"
	"path"
	"strings"
	"time"
)

// 首先实现往终端中输出日志

// 定义日志级别 Level
type Level int64

// 定义标志日志级别的常量
const (
	DEBGU Level = iota
	INFO
	WARNING
	ERROR
)

// 定义日志类别的结构体 Logger
type Logger struct {
	// 用来控制日志显示开关,只显示比指定的日志级别大的
	LogLevel Level  // 日志级别
	FilePath string // 日志文件路径
	FileName string // 日志文件名称
	MaxFile  int64  // 日志文件的大小
	FileObj  *os.File
}

// NewLog 构造函数用来生成指定的日志级别
func NewLog(l string, filepath string, filename string, maxfile int64) *Logger {
	// 根据用户初始化时候指定的日志级别,返回定义的常量
	level := WhichLevel(l)
	logg := Logger{
		LogLevel: level,
		FilePath: filepath,
		FileName: filename,
		MaxFile:  maxfile,
	}
	// 打开日志文件
	logg.LogOpenFile()
	return &logg
}

// 判断用户输入的记录日志级别
func WhichLevel(l string) Level {
	// 首先将用户输入的级别统一转化为小写
	l = strings.ToLower(l)
	switch l {
	case "debug":
		return DEBGU
	case "info":
		return INFO
	case "warning":
		return WARNING
	case "error":
		return ERROR
	default:
		return INFO
	}

}

func (l *Logger) LogOpenFile() {
	// 日志文件全路径
	fullpath := path.Join(l.FilePath, l.FileName)
	//日志的时间后缀
	now := time.Now()
	pose := fullpath + "." + now.Format("2006-01-02-15:04:05")
	// 打开文件
	fileobj, err := os.OpenFile(pose, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		panic(err)
	}
	l.FileObj = fileobj
}

// 传递的参数为类似printf的参数,可以传递格式化信息,页可以不传递
// Debug log
func (l *Logger) Debug(format string, a ...interface{}) {
	// 判断用户输入的日志级别,根据级别判断显示的日志
	if DEBGU >= l.LogLevel {
		logger := log.New(l.FileObj, "[ DEBUG ]", log.LstdFlags|log.Lshortfile)
		logger.Printf(format, a...)
	}
}

// Info log
func (l *Logger) Info(format string, a ...interface{}) {
	if INFO >= l.LogLevel {
		logger := log.New(l.FileObj, "[ INFO  ]", log.LstdFlags|log.Lshortfile)
		logger.Printf(format, a...)
	}
}

// Warning log
func (l *Logger) Warning(format string, a ...interface{}) {
	if WARNING >= l.LogLevel {
		logger := log.New(l.FileObj, "[WARNING]", log.LstdFlags|log.Lshortfile)
		logger.Printf(format, a...)
	}
}

// Error log
func (l *Logger) Error(format string, a ...interface{}) {
	if ERROR >= l.LogLevel {
		logger := log.New(l.FileObj, "[ ERROR ]", log.LstdFlags|log.Lshortfile)
		logger.Printf(format, a...)
	}
}
```

主函数中调用
```go
package main

import (
	"mylogger"
	"time"
)

func main() {
	logger := mylogger.NewLog("warning", "/Users/weiying/go", "testlog", 1024*1024*10)
	for i := 1; i <= 10; i++ {
		logger.Debug("this is Debug log")
		logger.Info("this is info log")
		id := 1000
		logger.Warning("this is warning log,id: %d", id)
		logger.Error("this is error log")
		time.Sleep(time.Second * 2)
	}
	defer logger.FileObj.Close()
}

```

上面的程序只是实现了日志的记录,下面来实现日志的切割

1. 按照文件大小切割
每次记录日志的之前,判断日志的大小,如果达到符合要求的大小,关闭这个文件,重新创建文件

```go

func (l *Logger) CheckFileSize() {
	fileinfo, err := l.FileObj.Stat()
	if err != nil {
		panic(err)
	}
	// 如果当前文件大小大于等于指定的值,说明要进行切割了
	if fileinfo.Size() >= l.MaxFile {
		//1. 关闭文件
		l.FileObj.Close()
		// 2. 备份文件
		os.Rename(path.Join(l.FilePath, fileinfo.Name()), path.Join(l.FilePath, fileinfo.Name())+".bak")
		// 3. 打开新的文件
		l.LogOpenFile()
		fmt.Println("切割")
	}
}
```

> 上面的方法需要在写入日志之前调用,判断日志的大小

2. 按照时间切割日志,原理与上面类似