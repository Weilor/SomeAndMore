# Flask源码阅读笔记（二）

## 从app.run()到@app.route发生了什么？

简而言之，app.run()启动了flask自己实现的一个web服务器，创建socket监听服务器端口，对接收到的客户端request进行解析，然后根据之前注册的路由把合法的请求内容传递给对应的路由处理函数，再从路由处理函数取得响应，加工成合法的http报文后返回给客户端程序。

还是以这个最小代码为例，如下：

```
from flask import Flask

app = Flask(__name__)


@app.route('/index')
def index():
    return "<h1>hello world!</h1>"

if __name__ == "__main__":
    app.run()
```

app.run()在脚本启动的时候开始执行，这个函数里面进行配置后调用了Werkzeug的方法然后启动了一个http server。配置主要为监听的IP地址和端口，如果没有通过入参传入就启用默认的127.0.0.1:5000配置进行监听。启动http server的时候创建了一个socket，用于监听接收客户端请求。当server的各种初始化和配置技术后，flask启动了serve_forever()方法，其实这就是一个循环，不断的从监听socket中取得request。

创建的http server时的主要工作除了创建一个socket监听端口外，还有绑定一个request handler。这个request handler将在server得到request的时候实例化，并按照http报文的格式将request请求分解，集合成environment交给app，按照path名字route rule交给对应的路由处理函数进行数据处理和响应。此外，http sever里还会对创建它的app进行反向指向，同时，request handler中也会回指创建它的server实例，并从server实例中取得app，如此，信息完成了一个从app到server到request handler再回到app的闭环线路。

上面这部分内容较为简单，之前接触过tcp/ip编程的话就不能理解。下面主要介绍一下这个闭环是如何指导request handler把请求交给对应的路由处理函数的。

这里涉及到的最关键的函数是request handler（具体是WSGIRequestHandler类）的run_wsgi()，这里简要介绍一下wsgi。

所谓wsgi，可以认为是一种web服务器程序和web应用程序间的约定或者规定，它规定了服务器程序和应用程序间数据传递的规则，当然这是一大块内容，我目前也没有全部研读过这些标准，但是简单来说，从web应用开发者的角度，它主要规定了web应用程序应当接受两个参数，一个是字典environment，一个是可调用的用来发送HTTP响应的函数，那么这个符合wsgi的应用程序（或者函数）应当长这个样子：

```
def app(environment,start_response)
```

这个程序应当被web server程序调用，并传入两个入参。

回到flask源码中看，这个体现wsgi规则的地方就是上面提到的run_wsgi()方法。这个方法还是挺长的，忘了提了，为了便于阅读理解，目前我看的都是0.10版本的flask源码。这个方法的前半部分就是解析http报文，把字符串形式的request信息，生成一个字典，也就是应用程序需要的那个environment入参；然后后半部分就是调用应用程序生成响应内容。

app的调用就是传入两个参数，和之前举的例子是不是一样？

（网站的网页是很多的，当然，虽然会很难以维护和理解，完全可以把所有网页内容响应内容都放在一个函数中实现，无非就是在函数中多做几个判断，区分客户端不同的请求内容。但这样做效率太低，所以才有使用web框架的意义。框架把大部分请求都会用到的部分提炼总结，变成一块公用代码，开发者只需要把每一个path对应的处理函数写好，注册到框架里，请求就可以完美处理了。这篇文字到上一段为止讲述的其实是flask自带的一个web服务器的功能，而flask本身作为web框架的工作其实下面才开始涉及。）

于是一个flask实例在这么一大圈后终于从request handler手中接过了接力棒，拿到了解析生成的environment信息，

```
k=wsgi.multiprocess v=False
k=SERVER_SOFTWARE v=Werkzeug/0.11.10
k=SCRIPT_NAME v=
k=REQUEST_METHOD v=GET
k=PATH_INFO v=/index
k=SERVER_PROTOCOL v=HTTP/1.1
k=QUERY_STRING v=
k=werkzeug.server.shutdown v=<function shutdown_server at 0x00000000036F8898>
k=CONTENT_LENGTH v=
k=HTTP_USER_AGENT v=Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.84 Safari/537.36
k=HTTP_CONNECTION v=keep-alive
k=SERVER_NAME v=127.0.0.1
k=REMOTE_PORT v=54515
k=wsgi.url_scheme v=http
k=SERVER_PORT v=5000
k=wsgi.input v=<socket._fileobject object at 0x00000000037112A0>
k=HTTP_HOST v=localhost:5000
k=wsgi.multithread v=False
k=HTTP_UPGRADE_INSECURE_REQUESTS v=1
k=HTTP_CACHE_CONTROL v=max-age=0
k=HTTP_ACCEPT v=text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
k=wsgi.version v=(1, 0)
k=wsgi.run_once v=False
k=wsgi.errors v=<open file '<stderr>', mode 'w' at 0x000000000276B150>
k=REMOTE_ADDR v=127.0.0.1
k=HTTP_ACCEPT_LANGUAGE v=zh-CN,zh;q=0.8
k=CONTENT_TYPE v=
k=HTTP_ACCEPT_ENCODING v=gzip, deflate, sdch
```

打出来看一下，发现environment里内容还真不少。

再接下来app会根据路由规则调用对应path的view function，产生response。这段内容涉及到上下文，是flask中比较关键的一部分内容。我继续看代码去咯。