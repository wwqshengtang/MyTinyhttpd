# 基于Linux的轻量级多线程 HTTP服务器


## 1. 项目框架图：
### 主要代码框架图
![](assets/16238295673459.jpg)

### 程序流程解析图
![](assets/16238295848291.jpg)

### socket 模型创建流程图
![](assets/16238319455001.jpg)

### 管道通信过程
![](assets/16238319765448.jpg)


## 2. 主要函数简略说明
- **int main(void)**   　　　　　　 
   主函数
- **int startup(u_short *)**; 　　　 　
  开启tcp连接，绑定监听套接字
- **void accept_request(void *arg)**; 　   
  每次收到请求，创建一个线程来处理接受到的请求
- **void serve_file(int, const char *)**;     
  如果不是CGI文件，直接读取文件返回给请求的http客户端　　　
- **void execute_cgi(int, const char *, const char *, const char *)**;
  执行cgi文件
- **int get_line(int, char *, int)**;  
  读取HTTP第一行数据
- **void headers(int, const char *)**;
  返回http头

将自己编写的 post.html 和 post.cgi 放入htdocs目录下，

在 accept_request 函数中将默认打开的index.html 文件修改为我们编写的 post.html 文件，可以显示网页信息，当提交表单执行POST操作后，会定位到 post.cgi 脚本执行

注意： 
- index.html 或 post.html必须没有执行权限，否则看不到内容，并且会产生Program received signal SIGPIPE, Broken pipe，因为程序中如果有可执行权限会当cgi脚本处理。所以假如html有执行权限先把它去除了，`chmod 600 index.html`
- .cgi 必须要有执行权限: `sudo chmod 755 post.cgi`

## 3. 执行
在文件目录下执行 make

![](assets/16238496183422.jpg)


运行后在浏览器中输入IP地址和选择的端口号： 127.0.0.1:54289，或10.211.55.4：54289
则页面为我们编写的 post.html 的内容

![](assets/16238496730781.jpg)


当我们填写这些信息并点击提交后，执行 post.cgi 脚本

![](assets/16238496871139.jpg)


## 4. 项目分析
### http 请求格式： 

![](assets/16238291455097.jpg)
http请求由三部分组成，分别是：起始行、消息报头、请求正文

起始行以一个方法符号开头，以空格分开，后面跟着请求的URI和协议的版本，格式如下：

```Method Request-URI HTTP-Version CRLF```

其中 Method表示请求方法；Request-URI是一个统一资源标识符；HTTP-Version表示请求的HTTP协议版本；CRLF表示回车和换行（除了作为结尾的CRLF外，不允许出现单独的CR或LF字符）

![](assets/16238325602006.jpg)


请求方法（所有方法全为大写）有多种，各个方法的解释如下：
- GET 请求获取Request-URI所标识的资源
- POST 在Request-URI所标识的资源后附加新的数据
- HEAD 请求获取由Request-URI所标识的资源的响应消息报头
- PUT 请求服务器存储一个资源，并用Request-URI作为其标识
- DELETE 请求服务器删除Request-URI所标识的资源
- TRACE 请求服务器回送收到的请求信息，主要用于测试或诊断
- CONNECT 保留将来使用
- OPTIONS 请求查询服务器的性能，或者查询与资源相关的选项和需求

![](assets/16238325160606.jpg)

> GET方法：在浏览器的地址栏中输入网址的方式访问网页时，浏览器采用GET方法向服务器获取资源
POST方法要求被请求服务器接受附在请求后面的数据，常用于提交表单

**http头部的 Content-Length：**

![](assets/16238309019906.jpg)

![](assets/16238428640241.jpg)
![](assets/16238429032207.jpg)
![](assets/16238429962735.jpg)
![](assets/16238430418351.jpg)


### Web、HTTP 和 CGI 之间的关系
TinyHTTPd 虽是迷你 Web 服务器程序，代码简短，完整涉及到 HTTP、TCP、CGI等基本协议，先来看一下这三者对于Web服务的作用。
Web客户端与服务器之间通过HTTP协议交互，传输层使用TCP协议来进行可靠性传输。（本篇不谈论安全协议，只分析最基本的Web服务框架）
CGI（Common Gateway Interface），也叫做通用网关接口。CGI是一种标准，它定义了动态文档应如何创建，输入数据应如何提供给应用程序，以及输出结果应如何使用。TinyHTTPd中用 Perl 写了CGI脚本程序，也可以用C/C++ 或 其他脚本语言替代

![](assets/16238318002108.jpg)

### 程序部分解读
对请求头进行处理，得到method、url
若为post，则调用cgi程序；
若为get，则需要判断是否带参数，若带参数，调用cgj程序；不带参数，则根据url获取path，并判断path是否存在，决定返回错误还是静态页面。

请求报文发送过来，需要getline()一行行读取以及recv()函数一字字读取；
而发送报文，需要通过send()函数一行行发送，包括状态行、响应正文等。

### Cgi程序详解：
Get:通过设置query_string环境变量，调用cgi；
Post:通过设置content_length环境变量以及获取请求报文主体数据，调用cgi；

Cgi程序放置在服务器上的一段可执行程序。作为HTTP服务器的时候，客户端可以通过GET或者POST请求来调用这可执行程序。

任何的HTML均是静态网页，它无法实现一些复杂的功能，而CGI可以为我们实现。CGI全称是 Common Gate Intergace ，在物理上，CGI是一段程序，它运行在Server上，提供同客户端 Html页面的接口。

如何调用cgi:
- GET：当使用这种方法时，CGI程序从环境变量QUERY_STRING获取数据。QUERY_STRING 被称为环境变量，就是这种环境变量把客户端的数据传给服务器。为了解释和执行程序，CGI必须要分析（处理）此字符串。
- POST：使用POST方法时，WEB服务器通过stdin(标准输入)，向CGI程序传送数据。服务器在数据的最后没有使用EOF字符标记，因此程序为了正确的读取stdin,必须使用CONTENT_LENGTH

### HTML程序通读
```HTML
<html>
	<head>
	<link rel="icon" href="data:;base64,=">
		<title>注册信息</title>
		<meta charset="utf-8">
	</head>
	<body>
		<form action="post.cgi" method="POST">
			账号：<input type="text" name="user" value="" size="10" maxlength="5">
			<br>
			<br>
			密码：<input type="password" value="" name="password" size="10">
			<br>
			<br>
			爱好：<input type="checkbox" name="tiyu" checked="checked">体育<input type="checkbox" name="singing">唱歌
			<br>
			<br>
			性别：<input type="radio" name="sex" checked="checked">男<input type="radio" name="sex">女
			<br>
			<br>
			自我介绍：<br>
			<textarea cols="35" rows="10" name="introduction">
				
			</textarea>
			<br>
			<br>
			地址：
			<select name="address">
				<option value="beijing">北京</option>
				<option value="sichuang">四川</option>
				<option value="shanghai">上海</option>
				<option value="wuhan">武汉</option>
			</select>
			<br>
			<br>
			<input type="submit" value="提交">
			<input type="reset" value="重置">
		</form>
	</body>
</html>
```
这是静态页面程序，当浏览器端输入服务器地址后，通过get方法发送给服务器，浏览器便可得到此程序，显示界面如下：

![](assets/16238329498561.jpg)

当填好上述信息后，点击按钮，则将参数通过post方法发送给服务器端的post.cgi程序并运行

**post.cgi程序解读：**

![](assets/16238330583908.jpg)

Post方法是将提交的数据放在发送报文的主体中，因此需要通过recv()函数获取，并通过write()函数管道通信给cgi程序中；而后cgi将上述的html程序通过管道通信给父进程，再通过send()函数形成发送报文的正文部分发送给浏览器

### 工作流程：
1. 服务器启动，在指定端口或随机选择端口绑定httpd服务
2. 收到一个 HTTP请求时（其实就是listen的端口accept的时候），派生一个线程运行 accept_request函数，
3. 取出 HTTP 请求中的 method(GET 或 POST)和 url，对于GET方法，如果有携带参数，则 query_string 指向 url中 ？后面的 GET参数
4. 格式化 url 到 path数组，表示浏览器请求的服务器路径，在TinyHttp中服务器文件是在htdocs文件夹下。当 url以 / 结尾，或 url 是一个目录，则默认在 path中加上 index.html（我们可以自己编写），表示访问主页
5. 如果文件路径合法，对于无参数的 GET 请求，直接输出服务器文件(index.html)到浏览器，即用 HTTP格式写到套接字上，跳到 (10)。其他情况(带参数 GET，POST方式，url为可执行文件) 则调用 execute_cgi 函数执行 cgi脚本
6. 读取整个 http请求并丢弃，如果是 POST 则找出 Content-Length，把 HTTP 200状态码写到套接字
7. 建立两个管道 cgi_output，cgi_input。并 fork 一个进程
8. 在子进程中，把 stdout 重定向到 cgi_output的写入端，把 stdin 重定向到 cgi_input 的读取端，关闭 cgi_input 的写入端和 cgi_output 的读取端。设置 REQUEST_METHOD 的环境变量，GET 的话设置 QUERY_STRING 的环境变量，POST 的话设置 CONTENT_LENGTH 的环境变量，这些环境变量都是为了给 cgi脚本使用，接着用 execl 运行cgi程序，
9. 在父进程中，关闭 cgi_input 的读取端和 cgi_output 的写入端，如果是 POST 的话，把 POST 数据写入 cgi_input，已被重定向到 stdin，读取 cgi_output 的管道输出到客户端，该管道输出是 stdout，接着关闭所有管道，等待子进程结束，
   
   ![](assets/16238423158208.jpg)


10. 关闭与浏览器的连接，完成了一次 HTTP请求与回应，因为 HTTP 是无连接的，


### 进程和线程
![](assets/16238432072580.jpg)
![](assets/16238434443926.jpg)
![](assets/16238436506436.jpg)
![](assets/16238439030909.jpg)
![](assets/16238439190827.jpg)



## 总结
- 项目开源：
- 应用技术： Linux， C++，Socket， TCP
- 项目描述： 此项目是基于Linux的轻量级多线程Web服务器，应用层实现了一个简单的 HTTP服务器，支持静态资源访问与动态消息回显(GET 和 POST方法)， 
- 主要工作
  - 支持迭代和多并发两种服务器模型
  - 实现 GET/POST 两种请求解析，支持 CGI脚本进行POST请求响应
  - 利用多线程机制提供服务，增加并行服务数量
  - 利用双管道进行不同进程间通信与数据交换，及时关闭无用管道
  - 实现快速地址再分配，避免紧急情况下服务器因宕机而引起的服务失效

  
