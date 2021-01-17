## 生成和解析 JWT

使用 jwt 需要使用`jwt-go`这个库来实现

```go
go get -u -v github.com/dgrijalva/jwt-go
```

1. 定制需求

根据我们的需求来决定 jwt 中保存哪些数据,比如说要保存用户名和密码的信息,那么我们要定义下面的结构体

```go
type MyCliams strcut {
  Username string `json:"username"`
  Password string `json:"password"`
  jwt.StandardClaims
}
```

自定义的结构体需要内嵌 jwt.StandardClaims 匿名字段,这个匿名字段中只包含了官方字段

2. 定义 jwt 的过期时间

```go
const TokenExpireDefaultTime = time.Hour * 2
```

3. 自定义签名

```go
var MySecret = []byte("夏天夏天悄悄过去")
```

4. 生成 JWT

```go
func GetToken(username ,password string)(string,error){
  // 创建一个我们自己的声明
  c := MyCliams{
    Username: username,
    Password: password,
    jwt.StandardClaims{
      ExpiresAt: time.Now().Add(TokenExpireDefaultTime).Unix(),
      Issuer: "weiying"
    },
  }
  //使用指定的签名方法创建签名对象
  token := jwt.NewWithClaims(jwt.SigningMethodHS256,c)
  // 使用指定的字符串签名后或得完整编码后的 token
  return token.SignedString(MySecret)
}
```

5. 解析 JWT

```go
func ParseToken(tokenString string)(*MyCliams,error){
  //解析
  token,err := jwt.ParseWithClaims(tokenString,&MyCliams{},func(token *jwt.Token)(i interface{},err error){
    return MyCliams,nil
  })
  if err != nil {
    return nil,err
  }
  // 校验 token
  if claims, ok := token.Claims.(*MyClaims); ok && token.Valid { // 校验token
		return claims, nil
	}
	return nil, errors.New("invalid token")
}
```



## 在gin中使用jwt



1. 首选注册获取token的路由

```go
r.POST("/login",authHandler)
```

2. 根据路由获取token

```go
func authHandler(c *gin.Context) {
	// 用户发送用户名和密码过来
	var user UserInfo
	err := c.ShouldBind(&user)
	if err != nil {
		c.JSON(http.StatusOK, gin.H{
			"code": 2001,
			"msg":  "无效的参数",
		})
		return
	}
	// 校验用户名和密码是否正确
	if user.Username == "q1mi" && user.Password == "q1mi123" {
		// 生成Token
		tokenString, _ := GenToken(user.Username)
		c.JSON(http.StatusOK, gin.H{
			"code": 2000,
			"msg":  "success",
			"data": gin.H{"token": tokenString},
		})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"code": 2002,
		"msg":  "鉴权失败",
	})
	return
}return
}
```

3. 使用其他接口的时候需要携带token,创建验证token的中间件

```go
// JWTAuthMiddleware 基于JWT的认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
	return func(c *gin.Context) {
		// 客户端携带Token有三种方式 1.放在请求头 2.放在请求体 3.放在URI
		// 这里假设Token放在Header的Authorization中，并使用Bearer开头
		// 这里的具体实现方式要依据你的实际业务情况决定
		authHeader := c.Request.Header.Get("Authorization")
		if authHeader == "" {
			c.JSON(http.StatusOK, gin.H{
				"code": 2003,
				"msg":  "请求头中auth为空",
			})
			c.Abort()
			return
		}
		// 按空格分割
		parts := strings.SplitN(authHeader, " ", 2)
		if !(len(parts) == 2 && parts[0] == "Bearer") {
			c.JSON(http.StatusOK, gin.H{
				"code": 2004,
				"msg":  "请求头中auth格式有误",
			})
			c.Abort()
			return
		}
		// parts[1]是获取到的tokenString，我们使用之前定义好的解析JWT的函数来解析它
		mc, err := ParseToken(parts[1])
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code": 2005,
				"msg":  "无效的Token",
			})
			c.Abort()
			return
		}
		// 将当前请求的username信息保存到请求的上下文c上
		c.Set("username", mc.Username)
		c.Next() // 后续的处理函数可以用过c.Get("username")来获取当前请求的用户信息
	}
}
```

4. 注册一个`/home`路由，发个请求验证一下吧。

```go
r.GET("/home", JWTAuthMiddleware(), homeHandler)

func homeHandler(c *gin.Context) {
	username := c.MustGet("username").(string)
	c.JSON(http.StatusOK, gin.H{
		"code": 2000,
		"msg":  "success",
		"data": gin.H{"username": username},
	})
}
```

如果不想自己实现上述功能，你也可以使用Github上别人封装好的包，比如https://github.com/appleboy/gin-jwt。

