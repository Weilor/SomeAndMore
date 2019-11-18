# Flask源码阅读笔记（一）

## flask的url route管理

定义flask实例的route时，使用一个装饰器来装饰函数，例如：

```
app = Flask(__name__)
@app.route('/')
def index():
    return "<h1>hello world!</h1>"
```

通过阅读这段代码，发现Flask在每个实例里定义了一个Map类（从werkzeug导入）url_map，一个字典view_functions。

注册路由分为两步。

第一步是通过一个Rule类作为入参调用Map类的add方法向url_map里添加一个路由规则，url_map里有一个列表_rules，用来存储实例下所有的路由规则，这个列表的每一个元素都是一个Rule类，其次，url_map中还有一个字典_rules_by_endpoint，这个字典也是存储路由规则的，不过它按照endpoint把它们分开存储了，key值就是endpoint，value是个Rule类。endpoint用来生成URL，可以是字符串、数字甚至是函数，这里使用的是注册的路由处理函数的名字。在向url_map添加路由规则的时候，会触发Rule类实例的绑定方法bind()，这个方法把url_map加入到Rule实例的成员变量中，并触发编译方法compile()，把rule规则和各路由选项解析生成正则表达式并存储进Rule实例中。

第二步是把需要注册的路由处理函数加入到Flask实例的字典view_functions中，key即为函数名。

到这里，flask生成了一个URL route系统，具体如何命中，继续研究代码。