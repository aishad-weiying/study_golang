# Beego 与数据的操作
在学习 Beego 操作数据库之前,先来复习一下,go 语言与数据库的操作

#### go 语言操作数据库

1. 首先安装数据库驱动
```bash
weiying@weiyingdeMacBook-Air go$ go get -u -v github.com/go-sql-driver/mysql
```

2. 导入数据库驱动程序
```go
import (
	"github.com/astaxie/beego"
	_ "github.com/go-sql-driver/mysql" 
)
```

3. 打开数据
```go
// 1. 打开数据库
		// 1.1 驱动名称，mysql，需要导入mysql驱动
// 1.2 连接参数, 账号:密码@(ip 地址:端口)/数据库名称?charset=使用的字符集
	sql.Open("mysql","root:admin123@tcp(172.19.36.53:3306)/goMysql?charset=utf8")
```

4. 操作数据库
```go
sql := "create table mysql_tb1(id int unsigned not null primary key,name varchar(20) not null,age tinyint unsigned)"
	_ , err = db.Exec(sql)
	if err != nil {
		fmt.Println(err)
		return
	}
```

5. 关闭数据库

完整代码如下func (c *MysqlController) ShowMysql() {
	// 1. 打开数据库
		// 1.1 驱动名称，mysql，需要导入mysql驱动
		// 1.2 连接参数, 账号:密码@(ip 地址:端口)/数据库名称?charset=使用的字符集
	db , err := sql.Open("mysql","root:admin123@(172.19.36.53:3306)/goMysql?charset=utf8")
	if err != nil {
		fmt.Println(err)
		return
	}
	// 关闭数据库
	defer db.Close()
	// 操作数据库
	sql := "create table mysql_tb1(id int unsigned not null primary key,name varchar(20) not null,age tinyint unsigned)"
	_ , err = db.Exec(sql)
	if err != nil {
		fmt.Println(err)
		return
	}
```go
func main() {
	// 1. 打开数据库
		// 1.1 驱动名称，mysql，需要导入mysql驱动
		// 1.2 连接参数, 账号:密码@(ip 地址:端口)/数据库名称?charset=使用的字符集
	db , err := sql.Open("mysql","root:admin123@(172.19.36.53:3306)/goMysql?charset=utf8")
	if err != nil {
		fmt.Println(err)
		return
	}
	// 关闭数据库
	defer db.Close()
	// 操作数据库
	sql := "create table mysql_tb1(id int unsigned not null primary key,name varchar(20) not null,age tinyint unsigned)"
	_ , err = db.Exec(sql)
	if err != nil {
		fmt.Println(err)
		return
	}
```

## Beego 中操作数据库
Beego 中操作数据库与上面没有什么太大的区别

1. 首先定义路由
当访问 mysql 资源的时候,使用 get 方法执行 ShowMysql 方法
```go
beego.Router("/mysql", &controllers.MysqlController{},"get:ShowMysql")
```

2. 定义 ShowMysql 方法
```bash
package controllers

import (
	"database/sql"
	"fmt"
	"github.com/astaxie/beego"
	_ "github.com/go-sql-driver/mysql"
	// 这个驱动我们不会用到，只有驱动会去调用，所以加上下划线
)

type MysqlController struct {
	beego.Controller
}

func (c *MysqlController) ShowMysql() {
	// 1. 打开数据库
		// 1.1 驱动名称，mysql，需要导入mysql驱动
		// 1.2 连接参数, 账号:密码@(ip 地址:端口)/数据库名称?charset=使用的字符集
	db , err := sql.Open("mysql","root:admin123@(172.19.36.53:3306)/goMysql?charset=utf8")
	if err != nil {
		fmt.Println(err)
		return
	}
	// 关闭数据库
	defer db.Close()
	// 操作数据库
	sql := "create table mysql_tb2(id int unsigned not null primary key,name varchar(20) not null,age tinyint unsigned)"
	_ , err = db.Exec(sql)
	if err != nil {
		fmt.Println(err)
		return
	}
	c.Ctx.WriteString("ok") // 执行成功的话,给浏览器返回一个字符串
}
```

1. 创建表的操作
```go
sql := "create table mysql_tb2(id int unsigned not null primary key,name varchar(20) not null,age tinyint unsigned)"
res , err := db.Exec(sql)
	if err != nil {
		fmt.Println(err)
		return
	}
beego.Info("执行结果：",res,err)
// 使用 beego.info 输出执行结果和错误信息
```

2. 插入数据的操作
```go
sql := "insert mysql_tb1(id,name,age) values (?,?,?)" //? 表示占位符
	res , err := db.Exec(sql,2,"jack",30) //将数据与占位符一一对应
	if err != nil {
		fmt.Println("数据插入错误:",err)
		return
	}
	count , _:= res.RowsAffected() // 查看受影响的数据的数量
	c.Ctx.WriteString("插入数据成功")
	c.Ctx.WriteString(strconv.Itoa(int(count)))
```

3. 查询数据

- 单行查询

```go
	sql := "select * from mysql_tb1 where id=1"
	// 获取一行数据
	row := db.QueryRow(sql)
	var id ,age int
	var name string
	//将获取到的数据保存在变量中
	row.Scan(&id,&name,&age)
	c.Ctx.WriteString(strconv.Itoa(id)+","+ name + ","+ strconv.Itoa(age))
```

- 查询多行数据
```go
	sql := "select * from mysql_tb1"
	// 获取多行数据
	rows , _ :=db.Query(sql)
	var id ,age int
	var name string
	//将获取到的每行数据循环保存在变量中
	for rows.Next() {
		rows.Scan(&id,&name,&age)
		c.Ctx.WriteString(strconv.Itoa(id)+","+ name + ","+ strconv.Itoa(age) + "\n")
	}
```

4. 删除数据
```go
// 删除数据
	sql1 := "delete from mysql_tb1 where id=2"
	_, err = db.Exec(sql1)
	if err != nil {
		beego.Info("删除数据错误：",err)
		return
	}
	c.Ctx.WriteString("删除数据成功\n")

	// 删除表
	sql2 := "drop table mysql_tb2"
	_ , err = db.Exec(sql2)
	if err != nil {
		beego.Info("删除表错误：",err)
		return
	}
	c.Ctx.WriteString("删除表成功\n")
```