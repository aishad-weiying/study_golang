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





