#### 获取用户传递数据的方法

- GetString(key string, def ...string) string :

	方法用来获取字符串类型的值,返回值为字符串类型

- GetStrings(key string, def ...[]string) []string :

	获取多个字符串类型的值,返回为字符串切片

- GetInt(key string, def ...int) (int, error) :

	方法用来获取整型数据,返回值为 int64 类型和错误类型

- GetInt8(key string, def ...int8) (int8, error)

- GetUint8(key string, def ...uint8) (uint8, error)

- GetInt16(key string, def ...int16) (int16, error)

....

- GetFloat(key string, def ...float64) (float64, error): 

	方法用来获取浮点型的数据

- GetFile(key string) (multipart.File, *multipart.FileHeader, error)：

	获取上传的静态文件
	参数为视图中 input 对应的 name

	返回值为: file 是上传的文件, header 是上传文件的文件头(包含文件的大小,名称等信息),error 是错误信息

> 注意 file 作为上传的文件,上传业务结束后需要关闭

- SaveToFile(fromfile, tofile string) error :存储用户上传的静态文件

	参数: fromfile 是视图 input 对应的 name,tofile 是在服务器端存储的文件路径 ,可以使用相对或绝对路径

	返回值:为错误信息

- GetFiles(key string) ([]*multipart.FileHeader, error) :

- GetBool(key string, def ...bool) (bool, error) 

	获取 bool 类型的数据,返回值为 bool 类型和错误信息

	作用: 接收前端传递过来的数据,无论是 get 请求还是 post 请求,都能接收

	参数: 一般是传递数据的 key 值,对应的是 form 表单中 input 标签的 name 属性

#### 返回值:
根据接收类型的不同,返回值类型也不同,最常用的 GetString() 方法只有一个返回值,如果没有接收到数据,就为空,其他的函数会返回错误类型,获取到的值一般是 input 标签中 value 的值,这个后面会写到

#### 还能获取通过正则路由传递的数据

 - Ctx.Input.Param(":ext")

get 请求传递的数据,也可以通过下面方式获取:

- Ctx.Input.Bind(":path")

例如
```go
url: http://127.0.0.1:8080/?userName=11&passwd=22

var name passwd string
c.Ctx.Input.Bind(&name,"userName") // 获取到为 11
c.Ctx.Input.Bind(&passwd,"passwd") //获取到的为 22
```