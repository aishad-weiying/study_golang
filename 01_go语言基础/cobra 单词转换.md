# cobra

一个用于生成命令行工具的框架(本身也是命令行工具)

非常的简单易用

目前 kubernetes,etcd,docker 等都是由 cobra 生成的命令行



### cobra 的安装

```go
go get -u github.com/spf13/cobra/cobra
```

使用`cobra`命令行(确保已经加到了环境变量中)

```go
cobra -h
```



### 创建一个 cobra 项目

使用 go modules 模式,首先开启 go modules 模式

```go
go env -w GO111MODULE=on
go env -w GOPROXY="https://goproxy.cn,direct"
```

创建一个项目

```bash
# 创建项目文件
mkdir clid
# 使用 go mod 初始化项目
go mod init
```

创建 cobra 项目

```bash
cd clid
# 使用 cobra 命令初始化项目
cobra init --pkg-name clid
# 查看初始化之后的目录结构
src:$ tree clid
clid
├── LICENSE
├── cmd
│   └── root.go
├── go.mod
└── main.go
```

### 查看默认生成的 root.go

root.go 文件主要分为以下几部分

1. command 结构体的声明

```go
var rootCmd = &cobra.Command{
	Use:   "clid",
	Short: "A brief description of your application",
	Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	// Uncomment the following line if your bare application
	// has an action associated with it:
	//	Run: func(cmd *cobra.Command, args []string) { },
}
```

- Use : root.go 中的 Use表示根命令命令
- Short: 表示命令的简单描述信息,在这个程序中,当我们执行 `./clid -h`的时候会显示出来的简要的提示信息
- Long : 表示命令的帮助信息
- Run : 执行这个命令的时候执行的函数,有两个参数,第一个参数是`*cobra.Command`类型的参数,用来获取命令参数,第二个参数是除了指定的参数之外的其他参数,保存在字符串切片中

2. 将所有的子命令都添加到根命令中

由 main.main()调用,这个函数只有在 root.go 中实现

```go
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

3. 初始化函数

```go
func init() {
	cobra.OnInitialize(initConfig)

	// Here you will define your flags and configuration settings.
	// Cobra supports persistent flags, which, if defined here,
	// will be global for your application.
	// 设置命令的参数标志位
	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.clid.yaml)")

	// Cobra also supports local flags, which will only run
	// when this action is called directly.
	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}
```

### 执行 clid 命令,查看执行的结果

```bash
clid:$ ./clid
# Long 参数指定
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:
# 授权信息
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

```

### 使用 cober 创建项目的时候指定授权信息个作者信息

```go
clid:$ cobra init -h

Flags:
  -h, --help              help for init
      --pkg-name string   fully qualified pkg name

Global Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -l, --license string   name of license for the project
      --viper            use Viper for configuration (default true)

```



### 为 clid 根命令指定子命令

有两张方法,第一种就是使用 `cobra add`命令添加,第二种就是直接创建 command.go 的文件,在文件中定义代码

第一种方式实际上就是指定生成command.go 的文件文件

1. 添加子命令

```bash
clid:$ cobra add log
log created at /Users/4paradigm/go/src/clid

```

2. 修改生成的 log.go 文件

```go
package cmd

import (

	"github.com/spf13/cobra"
	"log"
)

// logCmd represents the log command
var logCmd = &cobra.Command{
	Use:   "log", // 子命令
	Short: "查看日志", // 简要信息
	Long: `此命令用于查看clid程序运行的日志信息`,// 长的介绍信息
	Run: func(cmd *cobra.Command, args []string) { // 执行的函数
		log.Println("这是一条日志信息")
	},
}

func init() {
	rootCmd.AddCommand(logCmd) // 将此命令绑定到根命令
}
```

3. 运行命令查看

```bash
# 运行 clid 查看帮助信息
clid:$ ./clid -h
...
Available Commands:
  help        Help about any command
  log         查看日志 # 这里由Short 控制

# 查看子命令的帮助信息
clid:$ ./clid log -h
此命令用于查看clid程序运行的日志信息

Usage:
  clid log [flags]

Flags:
  -h, --help   help for log

# 运行命令
clid:$ ./clid log
2020/11/27 15:04:32 这是一条日志信息

```



### 为子命令添加子命命令

1. 创建子命令

```go
clid:$ cobra add version
version created at /Users/4paradigm/go/src/clid
```

2. 修改生成的 version.go 文件

```go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

// versionCmd represents the version command
var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "显示版本",
	Long: `显示log的版本信息`,
	Version: "1.0.0",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println(cmd.Version)
	},
}

func init() {
	logCmd.AddCommand(versionCmd) // 指定将此命令绑定到 log 子命令中
}
```

3. 运行命令

```bash
clid:$ ./clid log version
1.0.0
```

### 为命令添加选项

命令由选项和子命令组成,上面介绍了为命令添加子命令,下面介绍为命令添加选项

1. 以 log 为例,修改代码

```go
package cmd

import (

	"github.com/spf13/cobra"
	"log"
	"fmt"
)

// logCmd represents the log command
var logCmd = &cobra.Command{
	Use:   "log",
	Short: "查看日志",
	Long: `此命令用于查看clid程序运行的日志信息`,
	Run: func(cmd *cobra.Command, args []string) {
		// 获取选项以及选项传递的参数
		open ,err := cmd.Flags().GetBool("open")
		if err != nil {
			fmt.Println("获取open选项失败:",err)
			return
		}
		if open {
			level , err := cmd.Flags().GetString("level")
			if err != nil {
				fmt.Println("获取level选项失败:",err)
				return
			}

			log.Printf("这是一条[%s]日志信息\n",level)
		}else {
			log.Printf("日志未开启\n")
		}

	},
}

func init() {
	rootCmd.AddCommand(logCmd)
	// 设置字符串类型的选项，参数分别为：长参数  短参数 默认值 和 介绍
	logCmd.Flags().StringP("level","l","info","日志的级别：debug info error faild")
	logCmd.Flags().BoolP("open","o",true,"是否打开日志功能")
}

```

2. 执行

```bash
clid:$ ./clid log -l=error -o=false 
2020/11/27 15:25:48 日志未开启
clid:$ ./clid log -l=error -o=true
2020/11/27 15:25:55 这是一条[error]日志信息

```



### cobra 项目的实践

使用 cobra 实现的简单的单词模式转换项目:https://github.com/aishad-weiying/tour.git