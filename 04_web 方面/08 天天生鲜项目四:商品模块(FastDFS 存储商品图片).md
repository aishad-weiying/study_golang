## 商品模块
要实现商品模块的话,首先要完成后台界面,才能完成商品的添加和修改,那么可以使用之前的文章发布系统来实现这个后台管理

### 使用 fastDFS 完成添加分类的时候上传图片的存储
![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/9c1aff4ab64f1adf668923a3acf1dafe.png) 

```go
func UploadFile(this*beego.Controller,filePath string)string{

	//处理文件上传
	file ,head,err:=this.GetFile(filePath)
	if head.Filename == ""{
		return "NoImg"
	}

	if err != nil{
		this.Data["errmsg"] = "文件上传失败"
		this.TplName = "add.html"
		return ""
	}
	defer file.Close()

	//1.文件大小
	if head.Size > 5000000{
		this.Data["errmsg"] = "文件太大，请重新上传"
		this.TplName = "add.html"
		return ""
	}

	//2.文件格式
	//a.jpg
	ext := path.Ext(head.Filename)
	if ext != ".jpg" && ext != ".png" && ext != ".jpeg"{
		this.Data["errmsg"] = "文件格式错误。请重新上传"
		this.TplName = "add.html"
		return ""
	}
    // 连接 fastDFS,生成 client 对象
	client , err := fdfs_client.NewFdfsClient("/Users/weiying/go/client.conf")
	if err != nil {
		beego.Info("fdfs连接错误：",err)
		return ""
	}
  // 获取字节数组,用于保存文件内容,长度等于文件大小
	filebuffer := make([]byte,head.Size)
  // 将文件字节流写入到字节数组中
	file.Read(filebuffer)
    // 通过字节流上传文件
	res , err := client.UploadByBuffer(filebuffer,ext[1:])
	if err != nil {
		beego.Info("上传文件失败：",err)
		return ""
	}

	beego.Info("上传文件成功",res)


	return res.RemoteFileId
	
}
```

#### 连接 fastDFS 的配置文件
```go
connect_timeout = 20
network_timeout = 60
base_path = /home/fastdfs
tracker_server=172.19.36.69:22122
maxConns=100
use_connection_pool = false
connection_pool_max_idle_time = 3600
log_level = info
load_fdfs_parameters_from_tracker = false
use_storage_id = false
storage_ids_filename = storage_ids.conf
http.tracker_server_port = 80
```

## 首页数据展示

![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/c24bb9dbe0e2771088c6a62461ee312e.png) 

首页中主要分为四个部分,分别是商品类型的展示,,商品轮播图展示,促销商品展示和首页商品展示

1. 获取商品类型
```go
// 获取商品类型
	var goodsType []models.GoodsType
	db.QueryTable("GoodsType").All(&goodsType)
	this.Data["types"] = goodsType
```

2. 视图中展示商品类型
```go
<ul class="subnav fl">
    {{range .types}}
        <li><a href="#model01" class={{.Logo}}>{{.Name}}</a></li>
    {{end}}
</ul>
```

3. 获取首页商品轮播图数据
```go
// 获取轮播图数据
	var banner []models.IndexGoodsBanner
// 按照 index 排序,index 为显示顺序
db.QueryTable("IndexGoodsBanner").OrderBy("Index").All(&banner)
	this.Data["banner"] = banner
```

4. 视图中显示
```go
<ul class="slide_pics">
    {{range .banner}}
        <li><img src="http://172.19.36.69/{{.Image}}" alt="幻灯片"></li>
    {{end}}
</ul>
```

5. 获取促销商品数据
```go
// 获取促销商品
var IndexPromotionBanner []models.IndexPromotionBanner
db.QueryTable("IndexPromotionBanner").OrderBy("Index").All(&IndexPromotionBanner)
this.Data["IndexPromotionBanner"] = IndexPromotionBanner
```

6. 视图中系显示
```go
<div class="adv fl">
{{range .IndexPromotionBanner}}
    <a href="{{.Url}}"><img src="http://172.19.36.69/{{.Image}}"></a>
{{end}}
</div>
```

#### 获取首页商品
![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/c883edeefa835361d3165532b85f7ecb.png) 

通过上面的图片,可以看出来,首页商品展示的时候,需要展示不同类型的数据,这个应该是需要使用切片来保存不同类型数据的,相同类型的商品中,我们既要保存类型数据又要保存商品数据,那么应该使用 interface 类型,那么存储成 interface 类型的数据后,我们又没有办法区分具体是文字类型的数据,还是图片类型的数据,那么就要有个标识来指定,也就是使用 map 的 key 来充当标识,那么这个存放整体数据的容器的类型应该是**\[\]map\[string\]interface{}**


1. 首先定义存储容器
定义容器的时候,容器的长度应该是等于商品类型的个数,上面我们已经获取到了全部商品类型的切片,也就是这个切片的长度
```go
goods := make([]map[string]interface{},len(goodsType))
```

2. 将商品类型存储到容器中
```go
// 存储商品类型到容器中
for index , value := range goodsType {
    // value 为现有的类型数据，但是为了标示具体是什么数据，在容器中定义标识
    temp := make(map[string]interface{})
    temp["type"] = value
    goods[index] = temp
}
```

3. 下面存储图片商品和文字商品
```go
// 存储文字商品和图片商品
var goodsImage []models.IndexTypeGoodsBanner
var goodsText []models.IndexTypeGoodsBanner

for _ , temp := range goods {
    // 查询图片商品，赋值给结构体对象
    db.QueryTable("IndexTypeGoodsBanner").RelatedSel("GoodsSKU","GoodsType").Filter("GoodsType",temp["type"]).Filter("DisplayType",1).OrderBy("Index").All(&goodsImage)
		// 查询文字商品，赋值给结构体对象
	db.QueryTable("IndexTypeGoodsBanner").RelatedSel("GoodsSKU","GoodsType").Filter("GoodsType",temp["type"]).Filter("DisplayType",0).OrderBy("Index").All(&goodsText)
    // 将结构体对象添加到容器中
    temp["goodsText"] = goodsText
    temp["goodsImage"] = goodsImage
}

// 把容器传递给视图
this.Data["goods"] = goods
```

4. 视图中接收数据
```go
{{range .goods}}
    <div class="list_model">
        <div class="list_title clearfix">
            <h3 class="fl" id="model01">{{.type.Name}}</h3>
            <div class="subtitle fl">
                <span>|</span>
                {{range .goodsText}}
                    <a href="">{{.GoodsSKU.Name}}</a>
                {{end}}
            </div>
            <a href="#" class="goods_more fr" id="fruit_more">查看更多 ></a>
        </div>

        <div class="goods_con clearfix">
            <div class="goods_banner fl"><img src="http://172.19.36.69/{{.type.Image}}"></div>
            <ul class="goods_list fl">
                {{range .goodsImage}}
                    <li>
                        <h4><a href="#">{{.GoodsSKU.Name}}</a></h4>
                        <a href="#"><img src="http://172.19.36.69/{{.GoodsSKU.Image}}"></a>
                        <div class="prize">¥ {{.GoodsSKU.Price}}</div>
                    </li>
                {{end}}
            </ul>
        </div>
    </div>
{{end}}
```

## 商品详情页的展示

1. 商品详情页的请求路由
```go
<li>
    <h4><a href="#">{{.GoodsSKU.Name}}</a></h4>
    <a href="/goodsinfo?id={{.GoodsSKU.Id}}"><img src="http://172.19.36.69/{{.GoodsSKU.Image}}"></a>
    <div class="prize">¥ {{.GoodsSKU.Price}}</div>
</li>
```

2. 路由
```go
	beego.Router("/goodsinfo",&controllers.IndexController{},"get:ShowGoodsInfo")

```

#### 控制器
1. 获取 id 数据,并进行校验
```go
//获取id
id ,err := this.GetInt("id")
// 判断数据
if err != nil {
    beego.Info("传递的参数错误",err)
    this.Redirect("/",302)
    return
}
```

2. 数据处理
![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/f126a8f4231da40c2f8965dac70c9ad5.png) 

从上面的图片中,我们可以看出,这个页面可以用 layout 来实现

创建 goodslayout 页面
```go
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
        "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
    <meta http-equiv="Content-Type" content="text/html;charset=UTF-8">
    <title>天天生鲜-商品详情</title>
    <link rel="stylesheet" type="text/css" href="/static/css/reset.css">
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">

</head>
<body>
<div class="header_con">
    <div class="header">
        <div class="welcome fl">欢迎来到天天生鲜!</div>
        <div class="fr">
            {{if compare .username nil }}
                <div class="login_btn fl">
                    <a href="/login">登录</a>
                    <span>|</span>
                    <a href="/register">注册</a>
                </div>
            {{else}}
                <div class="login_btn fl">
                    欢迎您：<em>{{.username}}</em>
                    <span>|</span>
                    <a href="/user/logout">退出</a>
                </div>
            {{end}}
            <div class="user_link fl">
                <span>|</span>
                <a href="/user/userinfo">用户中心</a>
                <span>|</span>
                <a href="cart.html">我的购物车</a>
                <span>|</span>
                <a href="user_center_order.html">我的订单</a>
            </div>
        </div>
    </div>
</div>

<div class="search_bar clearfix">
    <a href="index.html" class="logo fl"><img src="images/logo.png"></a>
    <div class="search_con fl">
        <input type="text" class="input_text fl" name="" placeholder="搜索商品">
        <input type="button" class="input_btn fr" name="" value="搜索">
    </div>
    <div class="guest_cart fr">
        <a href="cart.html" class="cart_name fl">我的购物车</a>
        <div class="goods_count fl" id="show_count">1</div>
    </div>
</div>

<div class="navbar_con">
    <div class="navbar clearfix">
        <div class="subnav_con fl">
            <h1>全部商品分类</h1>
            <span></span>
            <ul class="subnav">
                <li><a href="#" class="fruit">新鲜水果</a></li>
                <li><a href="#" class="seafood">海鲜水产</a></li>
                <li><a href="#" class="meet">猪牛羊肉</a></li>
                <li><a href="#" class="egg">禽类蛋品</a></li>
                <li><a href="#" class="vegetables">新鲜蔬菜</a></li>
                <li><a href="#" class="ice">速冻食品</a></li>
            </ul>
        </div>
        <ul class="navlist fl">
            <li><a href="">首页</a></li>
            <li class="interval">|</li>
            <li><a href="">手机生鲜</a></li>
            <li class="interval">|</li>
            <li><a href="">抽奖</a></li>
        </ul>
    </div>
</div>
{{.LayoutContent}}
<div class="footer">
    <div class="foot_link">
        <a href="#">关于我们</a>
        <span>|</span>
        <a href="#">联系我们</a>
        <span>|</span>
        <a href="#">招聘人才</a>
        <span>|</span>
        <a href="#">友情链接</a>
    </div>
    <p>CopyRight © 2016 北京天天生鲜信息技术有限公司 All Rights Reserved</p>
    <p>电话：010-****888 京ICP备*******8号</p>
</div>
<div class="add_jump"></div>

<script type="text/javascript" src="js/jquery-1.12.2.js"></script>
<script type="text/javascript">
    var $add_x = $('#add_cart').offset().top;
    var $add_y = $('#add_cart').offset().left;

    var $to_x = $('#show_count').offset().top;
    var $to_y = $('#show_count').offset().left;

    $(".add_jump").css({'left': $add_y + 80, 'top': $add_x + 10, 'display': 'block'})
    $('#add_cart').click(function () {
        $(".add_jump").stop().animate({
                'left': $to_y + 7,
                'top': $to_x + 7
            },
            "fast", function () {
                $(".add_jump").fadeOut('fast', function () {
                    $('#show_count').html(2);
                });

            });
    })
</script>

</body>
</html>
```

3. 传递数据给商品
```go
// 根据id查询数据库
db := orm.NewOrm()
var goodsSKU models.GoodsSKU
goodsSKU.Id = id
//db.Read(&goodsSKU)
// 如果只单独的使用 read 查询的话.没有做关联查询
// 查询出来的结果中是没有关联表的数据的
	db.QueryTable("goodsSKU").RelatedSel("GoodsType","Goods").Filter("Id",id).One(&goodsSKU)

// 传递数据给视图
this.Data["goods"] = goodsSKU
```

4. 视图中展示数据
```go
<div class="breadcrumb">
    <a href="#">全部分类</a>
    <span>></span>
    <a href="#">新鲜水果</a>
    <span>></span>
    <a href="#">商品详情</a>
</div>

<div class="goods_detail_con clearfix">
    <div class="goods_detail_pic fl"><img src="http://172.19.36.69/{{.goods.Image}}"></div>

    <div class="goods_detail_list fr">
        <h3>{{.goods.Name}}</h3>
        <p>{{.goods.Desc}}</p>
        <div class="prize_bar">
            <span class="show_pirze">¥<em>{{.goods.Price}}</em></span>
            <span class="show_unit">单  位：{{.goods.Unite}}</span>
        </div>
        <div class="goods_num clearfix">
            <div class="num_name fl">数 量：</div>
            <div class="num_add fl">
                <input type="text" class="num_show fl" value="1">
                <a href="javascript:;" class="add fr">+</a>
                <a href="javascript:;" class="minus fr">-</a>
            </div>
        </div>
        <div class="total">总价：<em>16.80元</em></div>
        <div class="operate_btn">
            <a href="javascript:;" class="buy_btn">立即购买</a>
            <a href="javascript:;" class="add_cart" id="add_cart">加入购物车</a>
        </div>
    </div>
</div>

<div class="main_wrap clearfix">
    <div class="l_wrap fl clearfix">
        <div class="new_goods">
            <h3>新品推荐</h3>
            <ul>
                {{range .new2}}
                    <li>
                        <a href="/goodsinfo?id={{.Id}}"><img src="http://172.19.36.69/{{.Image}}"></a>
                        <h4><a href="/goodsinfo?id={{.Id}}">{{.Name}}</a></h4>
                        <div class="prize">￥{{.Price}}</div>
                    </li>
                {{end}}
            </ul>
        </div>
    </div>

    <div class="r_wrap fr clearfix">
        <ul class="detail_tab clearfix">
            <li class="active">商品介绍</li>
            <li>评论</li>
        </ul>

        <div class="tab_content">
            <dl>
                <dt>商品详情：</dt>
                <dd>{{.goods.Goods.Detail}}
                </dd>
            </dl>
        </div>

    </div>
</div>

```

5. goodslayout 中展示类型
```go
// 封装函数用来获取商品类型,传递给goodslayout
func ShowLayout(this *beego.Controller)  {
	// 获取所有的商品类型
	db := orm.NewOrm()
	var types []models.GoodsType
	db.QueryTable("GoodsType").All(&types)
	this.Data["types"] = types
	//获取用户是否登录的信息
	UserLoginCheck(this)
	// 指定layout
	this.Layout = "goodslayout.html"
}
// 展示商品的详情
func (this *IndexController) ShowGoodsInfo() {
	//获取id
	id ,err := this.GetInt("id")
	// 判断数据
	if err != nil {
		beego.Info("传递的参数错误",err)
		this.Redirect("/",302)
		return
	}

	// 根据id查询数据库
	db := orm.NewOrm()
	var goodsSKU models.GoodsSKU
	goodsSKU.Id = id
	db.Read(&goodsSKU)
	// 传递数据给视图
	this.Data["goods"] = goodsSKU

	// 调用封装的函数
	ShowLayout(&this.Controller)

	this.TplName = "detail.html"

}
```

6. goodslayout 中展示传递的数据
```go
<h1>全部商品分类</h1>
<span></span>
<ul class="subnav">
    {{range .types}}
        <li><a href="#" class="{{.Logo}}">{{.Name}}</a></li>
    {{end}}
</ul>
```

## 商品详情页的展示
![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/9b127cc3a475ad6eef3a7e80ab2eb802.png) 

1. 新品推荐,展示同类型数据的前两条商品
```go
// 获取同类型商品的最新两条信息
var goodsSKU2 []models.GoodsSKU
db.QueryTable("GoodsSKU").RelatedSel("GoodsType").Filter("GoodsType__Name",goodsSKU.GoodsType.Name).OrderBy("Time").Limit(2,0).All(&goodsSKU2)
this.Data["new2"] = goodsSKU2
```

2. 视图中展示
```go
<div class="new_goods">
    <h3>新品推荐</h3>
    <ul>
        {{range .new2}}
            <li>
                <a href="/goodsinfo?id={{.Id}}"><img src="http://172.19.36.69/{{.Image}}"></a>
                <h4><a href="/goodsinfo?id={{.Id}}">{{.Name}}</a></h4>
                <div class="prize">￥{{.Price}}</div>
            </li>
        {{end}}
    </ul>
</div>
```

## 商品的浏览历史
在用户中心中,会展示出5 个用户最近浏览的商品
![image.png](http://aishad.top/wordpress/wp-content/uploads/2020/06/221359ee300c397ca6a50c806750e6d6.png) 

下面将使用 redis 来存储这个浏览的历史记录,完成存之前要考虑一下几个问题

- 什么时候添加历史记录: 在用户登录的情况下,查看商品详情的时候添加
- 什么时候展示历史记录: 在用户的个人中心中显示
- 用什么格式来存储历史记录:
历史记录需要存储时哪个用户的浏览记录,而且还有先后顺序,那么我们使用 redis 的 list 来存储,存入的 key 值为 history_用户 id,value 为商品 id

1. 添加历史记录
```go
if username != nil {
		// 查询用户信息
		var user models.User
		user.Name = username.(string)
		db := orm.NewOrm()
		db.Read(&user,"Name")
		// 添加历史记录
		// 连接redis
		conn, err := redis.Dial("tcp","172.19.36.69:6379")
		if err != nil {
			beego.Info("redis连接失败",err)
			this.Redirect("/",302)
			return
		}
		defer conn.Close()
		//插入历史
		conn.Do("auth","admin123")
		// 如果多次浏览一个商品，只添加一次
		// 那么在插入之前，先把这个商品之前的记录在list中移除
		reply , err := conn.Do("lrem","histroy"+strconv.Itoa(user.Id),0,id)
		reply , _ = redis.Bool(reply,err)
		if reply ==  false {
			beego.Info("清除浏览历史失败")
		}
		_ , err = conn.Do("lpush","histroy"+strconv.Itoa(user.Id),id)
		if err != nil {
			beego.Info("插入失败2",err)
		}
	}
```

2. 展示历史记录
```go
var goods []models.GoodsSKU
// 连接redis
conn, err := redis.Dial("tcp","172.19.36.69:6379")
if err != nil {
    beego.Info("连接rediss失败",err)
}
defer conn.Close()
//查询数据
conn.Do("auth","admin123")
reply ,err := conn.Do("lrange","histroy"+strconv.Itoa(user.Id),0,4)
replyInts,_ := redis.Ints(reply,err)
for _,val := range replyInts{
    var temp models.GoodsSKU
    db.QueryTable("GoodsSKU").Filter("Id",val).One(&temp)
    goods = append(goods, temp)
}
this.Data["goods"] = goods
```

3. 视图中显示最近浏览商品
```go
<h3 class="common_title2">最近浏览</h3>
<div class="has_view_list">
    <ul class="goods_type_list clearfix">
        {{range .goods}}
        <li>
            <a href="/goodsinfo?id={{.Id}}"><img src="http://172.19.36.69/{{.Image}}"></a>
            <h4><a href="/goodsinfo?id={{.Id}}">{{.Name}}</a></h4>
            <div class="operate">
                <span class="prize">￥{{.Price}}</span>
                <span class="unit">{{.Price}}/{{.Unite}}</span>
                <a href="#" class="add_goods" title="加入购物车"></a>
            </div>
        </li>
        {{end}}
    </ul>
</div>
```