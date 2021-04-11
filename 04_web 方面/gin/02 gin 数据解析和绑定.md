## gin 数据解析和绑定

### json 格式数据的解析与绑定

```go
package main

import (
	"github.com/gin-gonic/gin"
)

type Login struct {
	// binding:"required" 表示这个字段为必须字段，不能为空
	Username string`form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string`form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func main() {
	// 1. 创建路由，返回Engine
	r := gin.Default()
	// 2. 绑定路由规则
	r.POST("/login",login)
	// 3. 监听端口，默认为8080
	r.Run(":80")
}

func login(c *gin.Context)  {
	// 创建结构体类型的数据，用来保存json格式的数据
	var j Login
	// 将request的body中的数据，自动按照json格式解析到结构体中
	err := c.ShouldBindJSON(&j)
	if err != nil {
		// gin.H是封装了json数据的工具
		c.JSON(400,gin.H{"status":"400","msg":err.Error()})
		return
	}
	// 判断用户名和密码是否正确
	if j.Username != "root" || j.Password != "admin" {
		c.JSON(400,gin.H{"status":"302"})
		return
	}
	c.JSON(400,gin.H{"status":"200","msg":"成功"})
}

```

使用命令行测试json 格式的解析与绑定
```bash
$ curl 127.0.0.1/login -H "content-type:application/json" -d "{\"user\":\"root\",\"password\":\"admin\"}" -X POST
{"msg":"成功","status":"200"}
```

### 表单数据的解析与绑定

```go
package main

import (
	"github.com/gin-gonic/gin"
)

type Login struct {
	// binding:"required" 表示这个字段为必须字段，不能为空
	Username string`form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string`form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func main() {
	// 1. 创建路由，返回Engine
	r := gin.Default()
	// 2. 绑定路由规则
	r.POST("/login",login)
	// 3. 监听端口，默认为8080
	r.Run(":80")
}

func login(c *gin.Context)  {
	// 创建结构体类型的数据，用来保存form格式的数据
	var f Login
	// 将request的body中的数据，自动按照form格式解析到结构体中,根据请求头部中的content-type自动推断
	err := c.Bind(&f)
	if err != nil {
		// gin.H是封装了json数据的工具
		c.JSON(400,gin.H{"status":"400","msg":err.Error()})
		return
	}
	// 判断用户名和密码是否正确
	if f.Username != "root" || f.Password != "admin" {
		c.JSON(400,gin.H{"status":"302"})
		return
	}
	c.JSON(400,gin.H{"status":"200","msg":"成功"})
}
```

html 表单代码
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>

<form action="http://127.0.0.1/login" method="post" enctype="application/x-www-form-urlencoded">
    用户名：<input type="text" name="username">
    <br>
    密&nbsp&nbsp码:<input type="password" name="password">
    <br>
    <input type="submit" value="登录">
</form>

</body>
</html>
```

### URI 的绑定与解析
```go
package main

import (
	"fmt"

	"github.com/gin-gonic/gin"
)

type Login struct {
	// binding:"required" 表示这个字段为必须字段，不能为空
	Username string `form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string `form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func main() {
	// 1. 创建路由，返回Engine
	r := gin.Default()
	// 2. 绑定路由规则,获取 url 中的参数
	r.GET("/login/:user/:password", login)
	// 3. 监听端口，默认为8080
	r.Run(":80")
}

func login(c *gin.Context) {
	// 创建结构体类型的数据，用来保存form格式的数据
	var f Login
	// 绑定url中的参数到结构体中
	err := c.ShouldBindUri(&f)
	fmt.Println("aaaa", f.Username, f.Password)
	if err != nil {
		// gin.H是封装了json数据的工具
		c.JSON(400, gin.H{"status": "400", "msg": err.Error()})
		return
	}
	// 判断用户名和密码是否正确
	if f.Username != "root" || f.Password != "admin" {
		c.JSON(400, gin.H{"status": "302"})
		return
	}
	c.JSON(400, gin.H{"status": "200", "msg": "成功"})
}
```

命令行中测试
```bash
$ curl http://127.0.0.1/login/root/admin
{"msg":"成功","status":"200"}
```