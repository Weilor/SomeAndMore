# Flask源码阅读笔记（四）

## flask的全局变量request

在上一个笔记中，提到了request context里有两个特殊的变量，request和session，在这个笔记中，结合flask源码，我们来看看在进行开发的时候，是怎么从上下文信息中取得这两个变量的。

这里要提到flask的全局变量，它们都定义在flask/globals.py模块中，如下：

```
# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```

种类上，request context和app context都是LocalStack类，这个类的初始化代码如下：

```
class LocalStack(object):    
    def __init__(self):
        self._local = Local()
```

它只有一个成员变量（其实还有一个property，top，用来返回栈顶元素），是个Local类，Local的初始化：

```
class Local(object):
    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)
```

它的实例里有两个元素，__storage__这个字典是存储空间，而__ident_func__和多线程有关，后面会涉及，先不讨论。

现在我们回过头来看LocalStack的入栈和出栈操作：

```
    def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()
```

那么可以看出，LocalStack类把数据存储在自己成员变量（Local）的中，在Local中绑定了一个stack属性。再看看Local的方法，发现它里面把__setattr__和__getattr__重载了。我们都知道，这两个内建函数是类实例设置和读取属性时调用的，它们被重载成了这个样子：

```
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}
```

所以，当给Local类添加一个属性的时候，其实就是在它的成员变量__storage__这个字典里增加一个键值对，键是indent，值是一个字典，也就是我们添加的属性名称和它的值。

另外四个全局变量，它们都是LocalProxy类，这个类又是个啥呢？首先看初始化的代码：

```
class LocalProxy(object):
    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
```

这里有两个成员，其中一个是一个私有成员_local，它存储了初始化时候的入参local。看到这里还是一头雾水，这样做怎么能把request从上下文中取出来呢？这个LocalProxy到底是个啥？

再来重新看一眼request的初始化过程：

```
request = LocalProxy(partial(_lookup_req_object, 'request'))
```

结合LocalProxy的初始化函数，那么这里request的私有变量_local就是这个偏函数了，这个偏函数的作用如下：

> ```
> def _lookup_req_object(name):
>     top = _request_ctx_stack.top
> if top is None:
> raise RuntimeError(_request_ctx_err_msg)
> return getattr(top, name)
> ```

其实就是从request context中取出request对象。那么从全局变量request到request context里的request是如何产生联系的？答案就在于LocalProxy也重载了__getattr__和__setattr__，看这里：

```
class LocalProxy(object):
……
    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    __setattr__ = lambda x, n, v: setattr(x._get_current_object(), n, v)

    def _get_current_object(self):
        """Return the current object.  This is useful if you want the real
        object behind the proxy at a time for performance reasons or because
        you want to pass the object into a different context.
        """
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)
```

当试图从LocalProxy设置或者取出属性的时候，这个操作会被_get_current_object捕获，并从私有变量_local中得到context中的对象（这里也就是request对象），并把需要的属性操作传递给它。如此一来，相当于给request context中的request对象穿了一个马甲，全局变量request会把操作导向给对应的request。

不得不说，这一段代码大量重载了内建函数，而且真有点绕，看了好久又结合了网上一些大神的文档才了解了个大概。