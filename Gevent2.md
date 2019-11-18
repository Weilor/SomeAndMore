# Python并发学习笔记：从协程到GEVENT（二）

昨天的笔记里对gevent的使用做了一点了解，今天做个实战演练，用flask和gevent实现一个支持long polling的“聊天室”。

想象一下，一个Web聊天室，上千人登录网页后开始在上面进行交流。消息提交是很简单的，就像表格那样实现就可以。但是页面刷新是一个需要着重思考的地方。该怎么实现页面的实时刷新呢？

由用户反复F5刷新肯定是个不好玩的方式，而C/S模型的特点就是所有的请求都由客户端发起，由服务端响应，由服务器主动推送也无法实现。这种情况下似乎有两种解决方法比较靠谱，一种就是长连接，一种就是长轮询。

长连接是在TCP层实现一个长时间保活的连接，这样服务器可以随时把消息推送给客户端。一般的HTTP交互过程中，客户端发起连接，服务器接收连接，两边一来一往，一个request结束的时候连接就断开了。这种设计是很有必要的，本身大部分HTTP服务没有对实时的需要，浏览器打开一个网页后还老保持和服务器的连接干嘛呢？客户端按需请求，服务器按需回复，连接没有长时间保持的价值。何况连接是有代价的，尤其是服务器端，能维护的连接是有限的，保持了一大堆连接，大部分情况却没有消息交互，这样浪费的事情谁也不想做。这就是长连接的最大弊端，资源占用高。

和长连接相比，长轮询的资源代价会小一点。客户端发起请求，服务器没有准备好数据的时候会先保持这个连接不断开，等到数据准备好的时候发送给客户端，客户端断开连接，处理数据。数据处理结束后再次发送请求。周而复始。这种情况下，当客户端拿到当前数据后连接就会中断，这样假如连接数已经饱和了，那其他的客户端就有机会拿到数据。

本文中实现的聊天室就是一个使用长轮询（Long Polling）实现实时消息交互的Web服务。

多人在线的场景需要考虑到并发的实现。长轮询中会尽量减少查询的次数，尽量做到客户端不需要频发发送查询请求。这样就意味着连接需要阻塞在数据获取的部分。那么一个单进程单线程的Web服务器就不够用了。想象一下，一个用户发送了请求后其他用户发现服务器不响应了，这得多让人崩溃。这里可以使用gevent提供的pwsgi服务器。

另一方面，客户端需要反复发起查询请求，同时还希望能够在不刷新网页的情况下实现页面数据的更新，那一个可行的方案就是使用JavaScript来保证这点。由脚本不断的发起请求，得到数据就刷新页面，然后继续发起请求。

思路还是很简单的，下面结合代码捋一遍。

```python
# coding = utf-8

from gevent.pywsgi import WSGIServer
from gevent.queue import Queue, Empty
from flask import Flask, render_template, redirect, url_for, request
from forms import LoginForm
from flask_bootstrap import Bootstrap
import simplejson

app = Flask(__name__)
bootstrap = Bootstrap(app)
app.config['SECRET_KEY'] = 'HARD'
messages = []
user_dict = {}


@app.route('/put_post/<user>', methods=["POST"])
def put_post(user):
    post = request.form['message']
    for u in user_dict:
         user_dict[u].put_nowait(post)
    return ""


@app.route('/get_post/<user>', methods=["GET", "POST"])
def get_post(user):
    q = user_dict[user]
    try:
        p = q.get(timeout=10)
    except Empty:
        p = ''
    return simplejson.dumps(p)


@app.route('/chatroom/<user>/')
def chatroom(user):
    user_con = user_dict.setdefault(user, Queue())
    return render_template("chatroom.html", user=user)


@app.route("/login", methods=["GET", "POST"])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        return redirect(url_for("chatroom", user=form.name.data))
    return render_template("login.html", form=form)


if __name__ == '__main__':
    http_server = WSGIServer(('127.0.0.1', 5000), app)
    http_server.serve_forever()
```

后端程序还是很简明的。我按照网页执行的顺序依次说明一下。后端启动一个由gevent提供的一个并发http服务器。用户首先访问的是login，在这里会看到一个表格，用来给自己取一个名字并传给后端服务器。服务器取得用户名后把页面重定向到chatroom方法，在这会把用户名和一个队列关联在一起作为全局变量user_dict这个字典的一个键值对，然后返回一个聊天室界面HTML。put_post和get_post分别是聊天信息写入和读入的处理方法。用户提交表单后，脚本重写了表单的提交方法，页面不会刷新，消息传入put_post中处理，put_post把消息写入每一个用户的队列中然后返回。脚本发送的查询请求由get_post处理，每一个用户都会不断的发送这个请求，请求处理执行到p = q.get(timeout=10)的时候被阻塞，因为消息队列中不一定有数据。当数据到达或者超时之后，方法返回，连接中断。

网页代码如下：

```html
<!--login.html-->

{% extends "bootstrap/base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block content %}
    {{ wtf.quick_form(form) }}
{% endblock %}
<!--chatroom.html-->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Chatroom</title>
    <script type="text/javascript" src="https://cdn.bootcss.com/jquery/3.0.0/jquery.min.js"></script>
    <script type="text/javascript">
    $(document).ready(function(){
      $('#form').submit(function(e) {
          $("#test").hide()
                var message = $("#post").val();
                $.ajax({
                    'type'     : 'POST',
                    'url'      : '/put_post/{{user}}',
                    'data'     : { 'message': message },
                    'dataType' : 'json',
                });
                e.preventDefault();
            });
      var longPoll = function(){
          return $.ajax({
              type: 'get',
              url:  '/get_post/{{ user }}',
              async: true,
              timeout: 20000,
              success: function(data){
                  if(data.length > 0){
                    $("#message").append($("<li>" + data + "</li>"))
                  }
                  return longPoll()
              },
              dataType : 'json'
          });
      };
    longPoll();
    });
    </script>
</head>
<body>
    <p id="test">there</p><br/>
    <form id="form">
        <input type="text" id="post" />
        <input type="submit" value="Submit" />
    </form>
    <ul id="message">
        {% for message in messages %}
            <li>{{ message }}</li>
        {% endfor %}
    </ul>
</body>
</html>
```

这是个比较简单的应用，但是可以说明一下long polling的实现思路和大体结构。总的来说，有gevent的参与后并发实现思路会很清晰，这个模块的功能强大，下面有时间的话要好好读读源码。

附上代码地址

[https://github.com/Weilor/flask-gevent-longpolling](https://link.zhihu.com/?target=https%3A//github.com/Weilor/flask-gevent-longpolling)

Measure

Measure