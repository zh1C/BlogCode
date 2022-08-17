---
author: "Narcissus"
title: "Golang实现博客(一)"
date: "2021-09-03"
description: "Go语言实现简单的博客系统笔记记录。"
tags: ["Golang", "Web"]
categories: ["Golang"]
---

## 基础准备

### 1、热加载

`air`是Go语言的热加载工具，它可以监听文件或目录的变化，自动编译，重启程序。大大提高开发期的工作效率。

- 安装使用`air`工具

在项目的根目录下使用命令`go get -u github.com/cosmtrek/air`进行安装，此时在go.mod文件中会自动生成模块依赖。

**注意：在Go Modules第三方依赖管理模式下，使用`go get -u 地址`来下载安装第三方依赖。**

- 使用`air init`会在根目录下生成一份配置文件，可以配置项目根目录，临时文件目录，编译和执行的命令等。
- 使用`air`命令就可以运行程序，后续会自动编译，启动程序，并监听当前目录中的文件修改。

![ScreenShot2021-08-30 13.01.31](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032554.png)

### 2、Web 一些散知识点

- `http.HandleFunc("/", handler)`里传参的`/`意味着任意路径。
- 可以通过`WriterHeader()`来设置响应的状态码。常见的有(StatusNotFound 404:页面没找到等)可以通过文档查看。

## 路由和中间件

### 1、路由

#### ServeMux和Handler

Go语言中处理HTTP请求主要跟两个东西有关：ServeMux和Handler。

ServeMux本质上是一个HTTP请求的路由器(或者叫多路复用器，Multiplexor)。他把收到的请求与一组预先定义的URL路径列表做对比，然后在匹配到路径的时候调用关联的处理器（Handler）。

**`http.ListenAndServe(addr string, handler Handler)`中handler通常是nil,此种情况下会使用DefaultServeMux**

#### http.ServeMux的对比其他第三方路由优缺点

优点：

1. 标准库意味着随着Go打包安装，无须另行安装
2. 测试充分；稳定、兼容性强
3. 简单、高效

缺点：

1. 缺少web开发常见的特性
2. 在复杂的项目中使用，需要写更多的代码。

常见的第三方路由有HttpRouter、gorilla/mux。HttpRouter是目前速度最快的路由器，且被框架Gin所采用。

#### 路由解析规则

- **精准匹配**指路由只会匹配准确指定的规则。
- **长度优先匹配**一般用在静态路由上（不支持动态元素如正则和URL路径参数），优先匹配字符数较多的规则。

以下面的为例：

![ScreenShot2021-08-30 14.49.39](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032555.png)

使用**长度优先匹配**规则的http.ServeMux会把除了`/about`这个匹配的以外的所有URI都是用`defaultHandler`来处理。

使用**精准匹配**的gorilla/mux会把以上两个规则精准匹配到两个链接，`/`为首页，`/about`为关于，除此之外都是404未找到。

### 2、依赖管理Go Modules

#### 弃用GOPATH

Go Modules出现的目的之一就是为了解决GOPATH的问题。

在GOPATH时代，Go源码必须放置在`GOPATH/src`下，抛弃GOPATH的好处就是能下任意地方创建Go项目。另外，GOPATH有非常落后的依赖管理系统。因为在执行`go get`时无法传达任何版本信息。无法保证所有人的依赖版本都一致。

#### Go Modules日常使用

- **初始化**

新项目，使用`go mod init`初始化生成`go.mod`文件。

- **Go Proxy**

国内访问外网受限，一般需要配合Go Proxy使用，防止`go get`获取源码包时花费时间过长。安装package的原则是先拉最新的release tag，若无tag则拉最新的commit。

**使用`go env -w ...`来修改Go相关的环境变量。**

- **go.mod**

每一次的`go get`会同时修改`go.mod`和`go.sum`文件。查看 `go.mod`代码：

![ScreenShot2021-08-30 15.31.07](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032556.png)

几个参数：

1. module ----- 我们的项目在Go Module里也算一个Module
2. go ----- 指定了版本要求，最低1.15
3. require ----- 项目所需依赖

- **go.sum**

`go.sum`文件保存着依赖包的版本和哈希值。直接依赖包、间接依赖包的哈希值都会被保存。

- **indirect**

回到`go.mod`中，可以看到`require`区块里有`// indirect`字样：此标志标明这个依赖包还未被使用，如果在代码的某个地方`import`到的话，这个标志会自动去除。

- **go mod tidy命令**

此命令做整理依赖使用，执行时会把未使用的module移除掉。

- **源码包的存放位置**

默认源码包存放于`GOPATH/pkg/mod`中。

- **清空Go Modules缓存**

使用`go clean -modcache`命令清空本地下载的Go Modules缓存。

- **下载依赖**

默认情况下，执行`go run`和`go build`命令时，Go会基于自动`go.mod`文件自动拉取依赖。Go Module也提供了`go mod download`命令下载项目所需依赖。

- **所有Go Module命令**

![ScreenShot2021-08-30 16.13.09](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032558.png)

### 3、URI中的斜杠

访问以下两个链接：

- ocalhost:3000/about
- localhost:3000/about/

可以看到有`/`的链接会报404错误。我们需要在URL进入Gorilla Mux路由解析之前，将后面的`/`去掉。使用中间件会因为执行顺序的问题，Gorilla Mux会先匹配路由，在执行中间件，故使用中间件依然会返回404。

**解决方法很简单，就是写一个函数把Gorilla Mux包起来，在这个函数中我们先对请求做处理，再传给Gorilla Mux解析**。但需要注意把首页URL的`/`排除在外。

## 表单提交

### 1、读取表单请求数据

`r.ParseForm()`由http包提供，从请求中解析请求参数，必须是执行完这段代码，后面使用`r.PostForm`和`r.Form`才能读取到数据，否则为空数组。

- Form:存储了post、put和get参数，在使用之前需要调用ParseForm方法。
- PostForm:存储了post、put参数，在使用之前需要调用ParseForm方法。

如果不想获取所有请求内容，而是逐个获取的话，无需使用`r.ParseForm()`可直接使用`r.FormValue()`和`r.PostFormValue()`方法。

### 2、模板文件语法

双层大括号`{{}}`是默认的模板界定符。用于在HTML模板文件中界定模板语法。模板语法都包含在`{{`和`}}`中。

- **{{.}}语句**

![ScreenShot2021-08-31 14.17.03](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032558-1.png)

`{{.}}`中的点表示当前对象。当我们传入一个结构体对象时，我们可以使用`.`来访问结构体的对应字段。同理，当我们传入的变量是`map`时，也可以在末班文件中通过`.`根据key来取值。

- **with关键字**

![ScreenShot2021-08-31 14.20.31](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032600.png)

语法如下：

![ScreenShot2021-08-31 14.22.14](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032601.png)

- **注释**

注释，执行时会忽略。可以多行，不能嵌套并且必须紧贴分界符。

![ScreenShot2021-08-31 14.25.20](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032602.png)

- **变量**

还可以在模板中声明变量，用来保存传入模板的数据或其他语句生成的结果。具体语法如下:

![ScreenShot2021-08-31 14.25.23](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032603.png)

其中`$variable`是变量的名字，在后续的代码中可以使用该变量。

- **移除空格**

有时会不可避免的引入空格或换行符，导致模板最终渲染结果不如预期，这种情况下可以使用移除空格语法。

![ScreenShot2021-08-31 14.25.33](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032604.png)

**注意：**`-`要紧挨`{{`和`}}`，同时与模板值之间需要使用空格分隔。

- **条件判断**

![ScreenShot2021-08-31 14.25.40](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032606.png)

- **range遍历**

range遍历有两种写法，其中`pipeline`的值必须是数组、切片、字典或者通道。

![ScreenShot2021-08-31 14.38.04](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032606-1.png)

- **修改默认的分界符**

Go标准库的模板引擎使用两对花括号作为标识，而许多前端框架（如`Vue`和`AngularJS`）也是用两队花括号作为标识符，同时使用会产生冲突，需要修改标识符，演示修改Go语言模板引擎的默认标识符。

![ScreenShot2021-08-31 14.42.34](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032608.png)

## 操作数据库

### 1、MySQL驱动

#### 操作MySQL数据库

使用Go操作MySQL等数据库，一般有两种方式：

- 一是利用database/sql接口，直接在代码里硬编写sql语句
- 二是利用ORM,具体一点就是GORM,以对象关系映射的方式在抽象地操作数据库。

#### MySQL驱动使用

选用[github.com/go-sql-diver/mysql](https://github.com/go-sql-driver/mysql)项目作为数据库驱动。使用`go get -u github.com/go-sql-driver/mysql`下载驱动。

下载后，在项目中引入：

![ScreenShot2021-08-31 15.48.40](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032609.png)

注意导入MySQL驱动时，在包路径前添加了`_`,这里我们使用了**匿名导入**的方式来加载驱动。

**为什么需要匿名导入？**

因为引入的是驱动，操作数据库时我们使用的是sql库里的方法，而不会具体使用到`github.com/go-sql-driver/mysql`包里的方法，当有未使用的包被引入时，Go编译器会停止编译。为了能让编译器正常运行，需要使用匿名导入来加载。

**当导入一个数据库驱动后，此驱动会自行初始化(利用init()函数)并注册自己到Golang的database/sql上下文中，随后我们就可以通过database/sql包提供的方法来操作数据库了。**

驱动里的`init()`代码如下：

![ScreenShot2021-08-31 15.55.09](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032610.png)

> 注意:  Go 语言中，为了使用导入的程序包，必须首先对其进行初始化。初始化始终在单个线程中执行，并且以程序包依赖关系的顺序执行。初始化每个包后，会优先自动执行 init() 函数，并且执行优先级高于主函数的执行优先级。

### 2、连接数据库

#### sql.DB连接池

`sql.DB`结构体是database/sql包封装的一个数据库操作对象，包含了操作数据库的基本方法，通常情况下理解为**连接池对象。**

一般而言，使用`sql.Open()`函数便可以初始化并返回一个`*sql.DB`结构体实例，只需要传入驱动名称及对应的DSN便可。

![ScreenShot2021-08-31 18.44.52](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032611.png)

> 注意，调用`sql.Open()`时，并未开始连接数据库，只是为连接数据库做好准备而已。所以一般都会跟一个`db.Ping()`来检测连接状态。

#### 连接池配置信息设置

推荐以下设置：

![ScreenShot2021-08-31 19.02.35](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032613.png)

- **SetMaxOpenConns最大连接数**

设置连接池最大打开数据库连接数，<=0表示无限制，默认为0。[实验表明](https://learnku.com/go/t/49809)，在高并发的情况下，将值设为大于 10，可以获得比设置为 1 接近六倍的性能提升。而设置为 10 跟设置为 0（也就是无限制），在高并发的情况下，性能差距不明显。

但需要注意不能超过数据库系统设置的最大连接数。否则会出现MySQL错误：

![ScreenShot2021-08-31 19.08.50](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032613-1.png)

- **SetMaxIdleConns空闲连接数**

设置连接池最大空闲数据库连接数，<=0表示不设置空闲连接数，默认为2。[实验表明](https://learnku.com/go/t/49809)，在高并发的情况下，将值设为大于 0，可以获得比设置为 0 超过 20 倍的性能提升。
这是因为设置为 0 的情况下，每一个 SQL 连接执行任务以后就销毁掉了，执行新任务时又需要重新建立连接。很明显，重新建立连接是很消耗资源的一个动作。设置空闲连接数，当有新任务进来时，直接使用这些随时待命的连接传输数据，以此达到节约资源，提高执行效率的目的。

- **SetConnMaxLifetime过期时间**

设置连接池里每一个连接的过期时间，过期会自动关闭。理论上来讲，在并发的情况下，此值越小，连接就会越快被关闭，也意味着更多的连接会被创建。设置的值不应该超过 MySQL 的 wait_timeout 设置项（默认情况下是 8 个小时）。

#### DSN

`DSN`全称是`Data Source Name`，表示数据库连接源，用于定义数据库的连接信息，不同数据库的DSN格式不同。可以使用`mysql.Config`创建MySQL的连接信息：

![ScreenShot2021-08-31 19.54.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032615.png)

### 3、数据库表结构

#### 新建数据库

有两种方式，一种是使用命令行创建数据库，一种是使用可视化工具（NaviCat）。

##### 命令行创建数据库

连接数据库通用格式：

![ScreenShot2021-08-31 20.01.25](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032615-1.png)

端口默认是`3306`,主机默认是`localhost`，如果默认可以不传输，使用命令`mysql -u root -p`即可认证成功进入命令行模式。

使用命令：`CREATE DATABASE goblog CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`创建数据库。编码使用`utf8mb4_unicode_ci`可以支持存储Emoji,另外支持大小写不敏感(ci是Case Insensitive的缩写)。

#### 建表

使用`Exec()`来执行创建数据库表结构的语句。一般使用`sql.DB()`中的`Exec()`来执行没有返回结果集的SQL语句。例如`INSERT`，`UPDATA`，`DELETE`等语句。语法如下：

![ScreenShot2021-09-01 13.12.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032617.png)

`Exec()`方法的第一个返回值实现了`sql.Result`接口的类型，`sql.Result`的定义如下：

![ScreenShot2021-09-01 13.13.34](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032618.png)

### 4、插入数据

多变量声明的方式和引入多个包使用`import(...)`一样。

![ScreenShot2021-09-01 13.32.41](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032619.png)

- **prepare语句**

在数据库安全方面，Prepare语句是防范SQL注入攻击有效且必备的手段，SQL注入的例子请见----[Golang MySQL驱动中的Prepare语句（防止SQL注入）](https://learnku.com/go/t/49692#0e50ee)。[SQL注入详解](https://www.cnblogs.com/myseries/p/10821372.html)

![ScreenShot2021-09-01 13.32.58](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032620.png)

- **sql.Stmt**

当我们执行：

![ScreenShot2021-09-01 13.50.54](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032621.png)

会使用会使用 SQL 连接向 MySQL 服务器发送一次请求，此方法返回一个 `*sql.Stmt` 指针对象，我们将其赋值到 `stmt` 变量里。`stmt `是 `statement` 的简写，是声明、陈诉的意思。可以理解为将包含变量占位符 `? `的语句先告知 MySQL 服务器端。

此时的stmt是一个指针变量，会占用SQL连接，我们需要对其进行关闭以释放SQL连接。**及时关闭SQL连接很有必要**，否则很快就会报错`ERROR 1040: Too many connections`。

- **stmt.Exec()**

Prepare只会生产stmt，真正执行请求需要调用`stmt.Exec()`。`stmt.Exec()` 的参数依次对应 `db.Prepare()` 参数中 SQL 变量占位符 `?`。返回一个`sql.Result`对象。

### 5、显示文章

#### Prepare模式

`QueryRow()`是可变参数的方法，语法如下：

![ScreenShot2021-09-01 15.29.46](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032622.png)

它的参数可以为一个或者多个。参数只有一个的情况下，我们称之为**纯文本模式(易被SQL注入)**，多个参数的情况下称之为 **Prepare 模式**。

之所以称之为 Prepare 模式是因为当多个参数的情况下，QueryRow() 封装了 Prepare 方法的调用，也就是说，下面这段代码：

![ScreenShot2021-09-01 15.30.09](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032624.png)

等同于：

![ScreenShot2021-09-01 15.30.18](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032625.png)

使用 `QueryRow()`的 Prepare 模式不仅保证了安全性，更能提升可读性。

> 关于Prepare模式和纯文本模式，需要注意两点：
>
> 1. 使用Prepare 模式会发送两个 SQL  请求到 MySQL 服务器上，而纯文本模式只有一个；
> 2. 在使用路由参数过滤只允许数字的情况下，可以放心使用纯文本模式无需担心 SQL 注入。

- **Scan（）方法**

`QueryRow() `会返回一个 `sql.Row` struct，紧接着我们使用**链式调用的方式**调用了 `sql.Row.Scan() `方法：

![ScreenShot2021-09-01 15.46.23](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032627.png)

需要注意的是，返回的` sql.Row `是个指针变量，保存有 SQL 连接。当调用 `Scan()` 时，就会将连接释放。所以在每次 **QueryRow 后使用 Scan 是必须的**。
**极力推荐这种链式调用的方式，养成好习惯以避免掉进 SQL 连接不够用的坑。**

