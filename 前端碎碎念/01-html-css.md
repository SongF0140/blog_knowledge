# 前言

> **鸣谢**：本文参考了「栗子」的教程，是我学习后二次整理的内容，在此致谢；若有错漏，皆因我个人水平有限，与「栗子」无关。

作为 C++ 后端，我对前端的心态经历过几个阶段：一开始觉得跟自己关系不大，能躲就躲；后来发现进了公司，指不定哪天就被拉去写页面，躲是躲不掉的；最后想明白，与其每次只学个半吊子，不如把它当成 Web 必不可少的一环，系统地补上。

我的体会是，光会一点基础 JS 是不够用的。现在前端早就工程化了，Vue、React 这些框架，没有点基础根本读不懂——连页面路径在哪配、怎么跳转都摸不清。所以要么不碰，要碰就认真学。

另外，学前端还有两个很实际的好处：

* 一个程序员最好既会一门静态语言（比如 C++），又会一门动态语言（比如 JavaScript），这样对"语言"这件事的理解会更完整。
* 会了前端之后，很多小项目就能自己独立接下来，不用处处依赖别人。

这份笔记的定位是"后端视角"，而不是"前端视角"：

* 前端教程的作者往往不知道后端人卡在哪——他觉得是常识、一带而过的地方，恰恰是我当初最懵的地方。
* 每个框架动辄几十个小时，后端人耗不起，所以这里只挑对后端最有用的部分讲，其余能省则省。

内容分五章：

* HTML / CSS：对后端来说不是重点，但不能完全不懂，作为第一章。
* JavaScript：重点，尤其是 ES6 之后的新语法，看不懂这些，框架代码基本读不下去。
* Vue2、Vue3、React：三选一，看你所在项目用哪个就学哪个。
  * 这三章会涉及 TypeScript、VueCli、Vuex、VueRouter、ElementUI、Vite、CreateReactApp、React、Redux、ReactRouter 等工具和库。
* jQuery：调研下来还有些老项目在用，作为可选章节。



# 第一章. HTML 与 CSS

HTML 全称 HyperText Markup Language（超文本标记语言），网页就是用它写的，它负责定义网页的**内容和结构**。

* HyperText 指用超链接把网页组织起来、互相联系起来。
* Markup 指用 `<标签>` 的方式，给内容赋予不同的功能和含义。

CSS 全称 Cascading Style Sheets（层叠样式表），负责描述网页的**表现和展示效果**。



## 1. HTML 元素

HTML 由一个个元素（element）组成，比如：

```html
<p>Hello, world!</p>
```

* 上面这一整段称为一个元素。
* `<p>` 和 `</p>` 分别叫起始标签和结束标签。
* 标签包起来的 `Hello, world` 是元素的内容。
* `p` 是预先定义好的标签，作用是把内容当作一个独立段落。

元素还可以带属性，比如：

```html
<p id="p1">Hello, world!</p>
```

* 属性一般是预先定义好的，这里的 `id` 用来给元素一个唯一标识。

元素之间可以嵌套，比如：

```html
<p>HTML 是一门非常<b>强大</b>的语言</p>
```

下面这种就是错误的嵌套写法：

```html
<p>HTML 是一门非常<b>强大的语言</p></b>
```

不含内容的元素叫空元素，比如：

```html
<img src="1.png">
<img src="1.png"/>
```

* `img` 用来展示图片，`src` 指明图片路径。
* 两种写法等价：HTML5 里空元素末尾的 `/` 可写可不写。



## 2. HTML 页面

前面说的都是单个元素，把它们组合起来，就是一份完整的 HTML 页面：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>测试页面</title>
  </head>
  <body>
    <p id="p1">Hello, world!</p>
    <img src="1.png">
  </body>
</html>
```

* `html` 是根元素，整个页面只有一个，包住其它所有元素。
* `head` 里放的是不用于展示的内容，比如 `title`、`link`、`meta`。
* `body` 里放的是真正展示给用户的内容，比如文本、图片、视频、音频等。



## 3. 常见元素

### 1) 文本

#### Heading

```html
<h1>1号标题</h1>
<h2>2号标题</h2>
<h3>3号标题</h3>
<h4>4号标题</h4>
<h5>5号标题</h5>
<h6>6号标题</h6>
```

#### Paragraph

```html
<p>段落</p>
```

#### List

无序列表（unordered list）：

```html
<ul>
    <li>列表项1</li>
    <li>列表项2</li>
    <li>列表项3</li>
</ul>
```

有序列表：

```html
<ol>
    <li>列表项1</li>
    <li>列表项2</li>
    <li>列表项3</li>
</ol>
```

多级列表：

```html
<ul>
    <li>
    	北京市
        <ul>
            <li>海淀区</li>
            <li>朝阳区</li>
            <li>昌平区</li>
        </ul>
    </li>
    <li>
    	河北省
        <ul>
            <li>石家庄</li>
            <li>保定</li>
        </ul>
    </li>
</ul>
```

#### Anchor

锚（超链接）：

```html
<a href="网页地址">超链接文本</a>
<a href="网页地址" target="_blank">在新标签页打开</a>
```

* `href` 指定跳转目标。
* `target` 指定打开方式，`_blank` 表示在新标签页打开，默认是当前页跳转。



### 2) 多媒体

#### Image

```html
<img src="文件路径">
```

`src` 有三种格式：

* 文件地址

* data URL，格式如下：

  ```
  data:媒体类型;base64,数据
  ```

* object URL，需要配合 JavaScript 使用

#### Video

```html
<video src="文件路径"></video>
```

#### Audio

```html
<audio src="文件路径"></audio>
```



### 3) 表单

#### 作用与语法

表单的作用：**收集**用户填写的**数据**，并把这些数据**提交给服务器**。

语法：

```html
<form action="服务器地址" method="请求方式" enctype="数据格式">
    <!-- 表单项 -->
    
    <input type="submit" value="提交按钮">
</form>
```

* `method` 有两种请求方式：
  * `get`（默认）：提交时数据跟在 URL 地址之后。
  * `post`：提交时数据放在请求体里。
* `enctype` 在 post 请求时指定请求体格式：
  * `application/x-www-form-urlencoded`（默认）
  * `multipart/form-data`
* 表单项里，只有带 `name` 属性的数据才会被发送给服务器。



#### 常见的表单项

文本框：

```html
<input type="text" name="username">
```

密码框：

```html
<input type="password" name="password">
```

隐藏框：

```html
<input type="hidden" name="id">
```

日期框：

```html
<input type="date" name="birthday">
```

单选：

```html
<input type="radio" name="sex" value="男" checked>
<input type="radio" name="sex" value="女">
```

多选：

```html
<input type="checkbox" name="fav" value="唱歌">
<input type="checkbox" name="fav" value="逛街">
<input type="checkbox" name="fav" value="游戏">
```

文件上传：

```html
<input type="file" name="avatar">
```

其它常用表单项：

```html
<!-- label 用来给表单项添加说明文字，点击文字也能聚焦到对应输入框 -->
<label>用户名：<input type="text" name="username"></label>

<!-- 下拉选择 -->
<select name="city">
    <option value="bj">北京</option>
    <option value="sh">上海</option>
</select>

<!-- 多行文本 -->
<textarea name="intro" rows="3"></textarea>
```

* 注意：文件上传时，表单的 `method` 必须是 `post`，且 `enctype` 必须是 `multipart/form-data`，比如 `<form action="..." method="post" enctype="multipart/form-data">`。



## 4. HTTP 请求

### 1) 请求组成

一个 HTTP 请求由三部分组成：

1. 请求行
2. 请求头
3. 请求体

可以直接用 telnet 程序来测试。

### 2) 请求方式与数据格式

#### get 请求示例

```
GET /test2?name=%E5%BC%A0&age=20 HTTP/1.1
Host: localhost
```

* `%E5%BC%A0` 是「张」经过 URL 编码后的结果。

#### post 请求示例

```
POST /test2 HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 21

name=%E5%BC%A0&age=18
```

`application/x-www-form-urlencoded` 格式的细节：

* 参数拆成名字和值，中间用 `=` 分隔。
* 多个参数用 `&` 分隔。
* 「张」这类特殊字符需要先 `encodeURIComponent()` 编码成 `%E5%BC%A0` 才能发送。



#### json 请求示例

```
POST /test3 HTTP/1.1
Host: localhost
Content-Type: application/json
Content-Length: 25

{"name":"zhang","age":18}
```



json 对象格式：

```
{"属性名":属性值}
```

属性值可以是：

* 字符串 ""
* 数字
* true、false
* null
* 对象
* 数组

json 数组格式：

```
[元素1, 元素2, ...]
```



#### multipart 请求示例

```
POST /test2 HTTP/1.1
Host: localhost
Content-Type: multipart/form-data; boundary=123
Content-Length: 125

--123
Content-Disposition: form-data; name="name"

lisi
--123
Content-Disposition: form-data; name="age"

30
--123--
```

* `boundary=123` 用来定义分隔符。
* 起始分隔符是 `--分隔符`。
* 结束分隔符是 `--分隔符--`。



#### 数据格式小结

客户端发送：

* 编码：
  * `application/x-www-form-urlencoded`：url 编码
  * `application/json`：utf-8 编码
  * `multipart/form-data`：每部分编码可以不同
* 表单只支持以 `application/x-www-form-urlencoded` 和 `multipart/form-data` 两种格式发送数据。
* 文件上传必须用 `multipart/form-data`。
* JS 代码可以支持任意格式发送数据。

服务端接收：

* 对 `application/x-www-form-urlencoded` 和 `multipart/form-data` 格式，C++ 后端接收方式比较统一，只要让结构体（或对象）的字段名和请求参数名一一对应即可。
* 对 `application/json` 格式，C++ 后端接收时要把请求体反序列化到结构体/对象（比如用 JSON 库 nlohmann/json 配合 DTO 结构体）。



### 3) session 原理

HTTP 本身无状态，但可以有会话：

* 无状态指请求之间互相独立，第一次请求的数据，第二次请求不能直接复用。
* 有会话指客户端和服务端都有技术能暂存数据，让数据在请求间共享。

服务端用 session 技术暂存数据。

存：

```
GET /s1?name=zhang HTTP/1.1
Host: localhost
```

取：

```
GET /s2 HTTP/1.1
Host: localhost
Cookie: JSESSIONID=560FA845D02AE09B176E1BC5D9816A5D
```

* 服务器第一次创建 session 时，会生成唯一的 session id，并通过响应头 `Set-Cookie: JSESSIONID=...` 发给浏览器。
* 浏览器之后每次向该服务器发请求，都会自动带上 `Cookie: JSESSIONID=...`，服务器凭这个 id 找到之前暂存的数据，从而实现"有会话"。



用 session 做身份验证：

```mermaid
sequenceDiagram
participant Client
participant L as LoginController
participant i as LoginInterceptor
participant Session
rect rgb(200, 223, 255)
Client ->> +L : 登录请求
L ->> L : 检查用户名，密码，验证通过
L ->> +Session : 存入用户名
Session -->> -L: 
L -->> -Client: 登录成功
end
rect rgb(200, 190, 255)
Client ->> +i : 其它请求
i ->> +Session : 获取用户名
Session -->> -i : 
i ->> i: 用户名存在，放行
i -->> -Client : 
end
```






### 4) jwt 原理

用 jwt 做身份验证：

```mermaid
sequenceDiagram
participant Client
participant L as LoginController
participant i as LoginInterceptor

rect rgb(200, 223, 255)
Client ->> +L : 登录请求
L ->> L : 检查用户名，密码，验证通过
L -->> -Client : 登录成功，返回token
end

rect rgb(150, 190, 155)
Client ->> +i : 其它请求，携带token
i ->> i : 校验token，校验无误，放行
i -->> -Client : 
end
```

生成 token：

```
GET /j1?name=zhang&pass=123 HTTP/1.1
Host: localhost
```

校验 token：

```
GET /j2 HTTP/1.1
Host: localhost
Authorization: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9._1-P_TLlzQPb1_lCyGwplMZaKQ8Mcw_plBbYPZ3OX28
```

* 实际项目里常写成 `Authorization: Bearer <token>`，Bearer 是约定的认证方案名。
* 和 session 的区别：session 把数据存在服务端，靠 cookie 里的 id 去查；jwt 把数据存在 token 本身（客户端），服务端只做签名校验、不存数据，更适合分布式系统。



## 5. CSS

CSS 全称 Cascading Style Sheets，负责描述网页的表现与展示效果。

CSS 代码有两种位置：

```html
<!-- 1. 写在 head 的 style 标签内 -->
<head>
    <style>
        p { color: blue; }
    </style>
</head>

<!-- 2. 写在独立的 css 文件中，用 link 引入 -->
<head>
    <link rel="stylesheet" href="样式文件路径">
</head>
```

### 1) 选择器

* type 选择器 —— 按标签名匹配（元素选择器）
* class 选择器 —— 按元素的 `class` 属性匹配
* id 选择器 —— 按元素的 `id` 属性匹配

配合下面的 html 看三种选择器怎么写：

```html
<p>普通段落</p>
<p class="title">标题段落</p>
<p id="p1">唯一段落</p>
```

```css
/* type 选择器：匹配所有 <p> 元素 */
p {
    color: blue;
}

/* class 选择器：用 .类名，匹配 class="title" 的元素 */
.title {
    font-weight: bold;
}

/* id 选择器：用 #id值，匹配 id="p1" 的元素 */
#p1 {
    background-color: yellow;
}
```

* 优先级：id 选择器 > class 选择器 > type 选择器（这也是"层叠"规则的一个体现）。

### 2) 属性和值

* background-color : red;
* ...
* display

### 3) 布局

和布局相关的 html 元素：

* `div` —— 块级容器，本身没有语义和样式，独占一行，常用来划分页面区域、配合 CSS 做布局。
* `template` —— 模板元素，内部内容**不会**被浏览器渲染，配合 JS 克隆节点来动态生成结构相同的元素（第二章案例会用到）。

div 配合 display 的简单示例：

```html
<style>
    .container {
        display: flex;          /* 弹性布局：子元素横向排列 */
        justify-content: space-between; /* 子元素之间平均分布 */
    }
    .item {
        background-color: lightblue;
    }
</style>

<div class="container">
    <div class="item">左</div>
    <div class="item">中</div>
    <div class="item">右</div>
</div>
```

* 对后端来说，布局能看懂、能微调就够了，不必深究——前端框架的组件库（如 ElementUI）都已经把布局封装好了。
