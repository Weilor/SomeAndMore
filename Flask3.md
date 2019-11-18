# Flask源码阅读笔记（三）

## flask的request context、request和session

上回书说到，request handler哼哧哼哧地把客户端发来经过WebServer转发的request解析生成environment，交给flask的实例进行处理。

当flask拿到environment后，它做的第一件事就是初始化一个request context实例。代码的作者是这么介绍这个request context的：

> The request context contains all request relevant information. It is
> created at the beginning of the request and pushed to the
> `_request_ctx_stack` and removed at the end of it. It will create the
> URL adapter and request object for the WSGI environment provided.

这个上下文实例中初始化了一个Werkzeug提供的request对象（它包含了这个request的所有环境信息），还绑定了一个用于处理URL的适配器。此外，在这里还将第一次出现一个概念，session。

request对象会在后面的代码中经常出现，“人如其名”，它是对客户端请求的对象化表达，方便在代码中进行操作。它的初始化也比较简单，只需要传入wsgi服务器调用时传入的environ-ment参数即可。URL的适配器用于从flask实例的url map中找到符合当前请求path的路由规则，这样我们就可以依靠路由规则中的endpoint找到在viewfunction中记录的处理函数。说实话我目前并不理解url_adapter的初始化中为什么会对environment信息进行绑定，明明已经有request存在并且回合url_adapter同时存在于context中，需要进一步阅读和思考，但无论如何，adapter在这里初始化并且绑定了environment和flask的url map实例。接着context实例调用url_adapter的match方法，遍历url map中的路由规则，并使用规则自带的正则表达式对environment中的path info进行匹配，一旦得到匹配就会返回命中的路由规则。到这一步，终于可以把request请求传递给它对应路径的处理函数了，原理就是按照路由规则中的endpoint，在viewfunction中找到与它同名的function，把request实例和从客户端请求path info中解析出来的参数传递给它。

初始化完成后，request context会被压入全局的request context 栈中，在flask中有两个全局栈，一个就是这个request context栈，还有一个app的全局栈。在request context入栈的同时，会检查当前app的全局栈的栈顶是否是当前app的上下文信息，如果不是或者栈顶不存在的话，会初始化一个app的上下文信息并压入全局app上下文栈中。app的context实例初始化的时候，主要包括绑定app实例，初始化一个url adapter（使用的入参为空），初始化一个全局变量g，还有一个引用计数。

request context和app context入栈后，request context还会初始化一个session。session的生成要基于cookie，从request中取出cookie，并根据配置中的SECRET_KEY反序列化，得到负载并初始化一个CallbackDict，dict这个就是flask的session。

这部分代码中其实有许多地方还没有提到，比如blueprint，比如before request和after request，日后补上，记一个TODO。当然，程序员的TODO就是……哈哈哈