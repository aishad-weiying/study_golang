## 视图布局
平时浏览网页的时候,网站的几个页面之间的格式设计很类似,那么这种类似的页面都会用一些公用的组件,像导航栏和底部这种,这就是视图布局的作用

视图布局本质上上的两个 html 页面的拼接,比如我们现在有一个包含头部和尾部的 HTML 界面 layout.html,还有一个只包含添加文章的业务的界面,我们可以实现这两个页面的拼接

#### 使用

1. 控制器中代码
```go
c.Layout = "layout.html"  // 指定要拼接的页面
c.TplName = "index.html"  // 指定视图
```

2. 在对应的拼接的页面中要加入下面的代码,才能显示指定的视图
```go
{{.LayoutContent}}
```

### 代码中使用 layout 来实现页面拼接
1. 新建一个 layout.html 页面,只保留框架性的内容
```go
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>后台管理页面</title>
    <link rel="stylesheet" type="text/css" href="/static/css/reset.css">
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <script type="text/javascript" src="/static/js/jquery-1.12.4.min.js"></script>

</head>
<body>

<div class="header">
    <a href="#" class="logo fl"><img src="/static/img/logo.png" alt="logo"></a>
    <a href="/logout" class="logout fr">退 出</a>
</div>

<div class="side_bar">
    <div class="user_info">
        <img src="/static/img/person.png" alt="张大山">
        <p>欢迎你 <em>李雷</em></p>
    </div>

    <div class="menu_con">
        <div class="first_menu active"><a href="javascript:;" class="icon02">文章管理</a></div>
        <ul class="sub_menu show">
            <li><a href="/article/showarticle" class="icon031">文章列表</a></li>
            <li><a href="/article/addarticle" class="icon032">添加文章</a></li>
            <li><a href="/article/addarticletype" class="icon034">添加分类</a></li>
        </ul>
    </div>
</div>
{{.layoutcontent}}
</body>
</html>
```

2. index.html 页面只保留内容,去除框架内容
```go
<div class="main_body" id="main_body">
    <div class="breadcrub">
        当前位置：文章管理>文章列表
    </div>
    <div class="pannel">
        <span class="sel_label">请选择文章分类：</span>
        <form id="form" method="get" action="/article/showarticle">
            <select name="select" id="select" class="sel_opt">
                {{range .type}}
                    {{if compare .TypeName $.typeName}}
                        <option selected="true">{{.TypeName}}</option>
                    {{else}}
                        <option>{{.TypeName}}</option>
                    {{end}}
                {{end}}
            </select>
        </form>

        <table class="common_table">
            <tr>
                <th width="43%">文章标题</th>
                <th width="10%">文章内容</th>
                <th width="16%">添加时间</th>
                <th width="7%">阅读量</th>
                <th width="7%">删除</th>
                <th width="7%">编辑</th>
                <th width="10%">文章类型</th>
            </tr>

            {{range $index , $value := .article}}

                <tr>
                    <td>{{$value.Title}}</td>
                    <td><a href="/article/showarticlecontent?articleId={{$value.Id}}">查看详情</a></td>
                    <td> {{$value.Time}}</td>
                    <td>{{$value.Count}}</td>
                    <td><a href="/article/deletearticle?articleId={{$value.Id}}" class="dels" class="dels">删除</a></td>
                    <td><a href="/article/updatearticle?articleId={{$value.Id}}">编辑</a></td>
                    <td>{{$value.ArticleType.TypeName}}</td>
                </tr>
            {{end}}
        </table>

        <ul class="pagenation">
            {{ if compare .FirstPage true }}
            {{ else }}}
                <li><a href="/article/showarticle?pageindex=1&select={{.typeName}}">首页</a></li>
                <li><a href="/article/showarticle?pageindex={{.index | BeferPage}}&select={{.typeName}}">上一页 </a></li>
            {{end}}
            {{ if compare .LastPage true }}
            {{else}}
                <li><a href="/article/showarticle?pageindex={{.index | AfterPage}}&select={{.typeName}}">下一页</a></li>
                <li><a href="/article/showarticle?pageindex={{.pageNum}}&select={{.typeName}}">末页</a></li>
            {{end}}

            <li>共{{.num}}条记录/共{{.pageNum}}页/当前{{.index}}页</li>
        </ul>
    </div>
</div>
```

3. 控制器中实现视图布局
```go
	c.Layout = "layout.html"
	c.TplName = "index.html"
```

## LayoutSections
当一个 LayoutContent 不能满足使用的时候,可以通过LayoutSections 在 layout 界面中设置多个 section 

比如要是现在所有的使用 layout 的界面实现某个 js 触发器的功能,就可以在 layout 的界面中添加一个 section

### 实现

1.  新加一个 section 的 HTML 页面 indexJs.html
```go
<script type="text/javascript">
    window.onload = function (ev) {
        //找标签  触发了什么事件    实现什么功能
        $(".dels").click(function () {
            if(!confirm("是否确认删除？")){
                return false
            }
        })
        $("#select").change(function () {
            //让form发个请求
            $("#form").submit()
        })
    }
</script>
```

2. 新加一个 section 的 HTML 页面 head.html
```go

<body>

<div class="header">
    <a href="#" class="logo fl"><img src="/static/img/logo.png" alt="logo"></a>
    <a href="/logout" class="logout fr">退 出</a>
</div>

<div class="side_bar">
    <div class="user_info">
        <img src="/static/img/person.png" alt="张大山">
        <p>欢迎你 <em>李雷</em></p>
    </div>

    <div class="menu_con">
        <div class="first_menu active"><a href="javascript:;" class="icon02">文章管理</a></div>
        <ul class="sub_menu show">
            <li><a href="/article/showarticle" class="icon031">文章列表</a></li>
            <li><a href="/article/addarticle" class="icon032">添加文章</a></li>
            <li><a href="/article/addarticletype" class="icon034">添加分类</a></li>
        </ul>
    </div>
</div>
</body>>
```
3. 控制器中设置 layout 和 section
```go
    c.Layout = "layout.html"                      // layout 界面
	c.LayoutSections = make(map[string]string)    // 定义section
	c.LayoutSections["indexJs"] = "indexJs.html"  // 指定页面
    c.LayoutSections["headJs"] = "head.html"
	c.TplName = "index.html"
```

4. layout 页面中指定使用 section 页面
使用 {{.section 名称}}
```go
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>后台管理页面</title>
    <link rel="stylesheet" type="text/css" href="/static/css/reset.css">
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <script type="text/javascript" src="/static/js/jquery-1.12.4.min.js"></script>
    {{.indexJs}}
</head>
{{.headJs}}
{{.LayoutContent}}

</html>
```