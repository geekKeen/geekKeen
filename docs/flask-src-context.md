# 深入 Flask 源码理解 Context


---

## Flask 中的上下文对象

知乎问题 [编程中什么是「Context(上下文)」](https://www.zhihu.com/question/26387327) 已经能够简单地说明什么是 Context，它是一个程序需要的外部对象，类似于一个全局变量。而这个变量的值会根据提供的值而改变。

Flask 中有分为请求上下文和应用上下文：

|对象|Context类型|说明|
|:---:|:---|:---|
|current_app|AppContext|当前的应用对象|
|g|AppContext|处理请求时用作临时存储的对象|
|request|RequestContext|请求对象，封装了Http请求的内容|
|session|RequestContext|用于存储请求之间需要记住的值|

> Flask 分发请求之前激活程序请求上下文，请求处理完成后再将其删除。

Flask 中的 Context 是通过栈来实现。


---

## Flask 的 Context 实现

Flask 的核心功能依赖于 Werkzeug 库。

### _app_ctx_stack & _request_ctx_stack

这两种栈定义在 `flask/global.py` 中。

```python
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
```

首先需要了解一下 Werkzeug 中关于 `LcoalStack` 的相关内容。

`Local` 类

`Local` 是定义了一个 `__storage__` 字典，其中的键为 `thread` 的 `id` 值。
```python
class Local(object):
    __slots__ = ('__storage__', '__ident_func__')
    
    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)
        
    def __setattr__(self, name, value):
        ident  = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            raise AttributeError(name)
    ...
```

`LocalStack` 类

`LocalStack` 则内部维护一个 `Local` 实例。主要的作用是将 `Local` 维护的 `__storage__` 字典中键为  `__ident_func__()` 对应的值定义为 `stack`。

```python
class LocalStack(object):
    def __init__(self):
        self._local = Local()
        
    def push(self, obj):
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv
        
    def pop(self, obj):
        pass
```

`LocalProxy` 类

`LocalProxy`类是一个代理类，应用到设计模式当中的[代理模式](http://dongweiming.github.io/python-proxy.html)。简单地讲，我们不需要去了解当前的环境，而直接去操作这个 `Proxy` 类，这个 `Proxy` 类会将所有的操作反馈给正确的对象。

```python
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__')
    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
    
    def _get_current_object(self):
        # 通过此方法获取被代理的对象
        if not hasattr(self.__local, '__release_local__')
            return self.__local
        try:
            return gerattr(self.__local,self.__name__)
        except Attribute:
            raise RuntimeError('no object bound to %s' % self.__name__)
    ...
    # 其他操作
```

### request & RequestContext

Flask 源码中关于 `request` 的定义：

```python
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)
    
request = LocalProxy(partial(_lookup_req_object, 'request'))
```

从源码可以看出，`request` 是 `_request_ctx_stack` 栈顶元素的一个属性。实际上 `_request_ctx_stack` 栈中的元素是 `ReuqestContext` 对象的实例， 而 `ReuqestContext` 中包含了 `request` 请求的所有信息,包括 `Session` 信息。

```python
class ReuqestContext(object):
    def __init__(self, app, environ, request=None):
        if reuqest is None:
            request  = Request(environ)
        self.requst = request
        self.app = app 
        self.session = None
        ...
        # 这个列表包含了与 request 相关联的 Application
        self._implicit_app_ctx_stack = []
        self.match_request()

    def push(self, object):
        """
        这里需要实现的是：当 RequestContext push 到
        _request_ctx_stack 时， 需要检测是否有对应的
        AppContext。如果没有，则会将当前 self.app push
        到 AppContext 中，同时将self.app 加入
        _implicit_app_ctx_stack 列表中； 否则
        _implicit_app_ctx_stack 添加 None。
        """
        pass
        
    def pop(self):
        """
        当 ReuqestContext 弹出 _request_ctx_stack 的
        方法。注意：request 清理之后的动作。如执行
        teardown_request。
        """
        pass
```

这里传入的 `app`，就是 `Flask` 的程序实例。 
`RequestContext` 实例的创建在 `Flask` 类方法中。

```python
class Flask(_PackageBoundObject):
    ...
    request_class = ReuqestContext
    def wsgi_app(self, environ, start_response):
        ctx = self.request_class(environ)
        ctx.push
        ...
        
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```

Flask 中 `Request` 对象继承了 Werkzeug 中的 `Request` 对象。
上述代码涉及到 `WSGI`，它强调 Appication 必须是一个可调用对象。
后期的工作之一是了解 `WSGI`。

### Session

在 session.py 文件中定义了 有关Session的内容。Flask 中 Session 是构建在 Cookie 上面的。其中定义了关于 `Session` 的接口。
 
```python
class SessionMixin(object):
    """定义了Session的最小属性"""
    
class SecureCookieSession(CallDict, SessionMixin):
    """ CallDict 是 werkzeug 中的数据结构 """

class NullSession(SecureCookieSession):
    """ 定义了空 session 结构 """
    
class SessionInterface(object):
    """ 定义了 Session接口的属性，依赖于 app.config 
    中的信息。同时，规定了只要是继承SessionInterface
    必须实现 open_session 和 save_session 方法
    """
class SecureCookieSessionInterface(SessionInterface):
    """ 
    主要是实现了 open_session 和 save_session 方法
    """
```

如下代码则是 `session` 的应用。

```python
# flask/app.py
class Flask(_PackageBoundObject):
    session_interface = SecureCookieSessionInterface()
    def open_session(self, request):
        return self.session_interface.open_session(self, request)
        
    def save_session(self, session, response)
        return self.session_interface.save_session(\
            self, session, response)
            
    def process_response(self, response):
        ctx = _request_ctx_stack.top
        ...
        if not self.session_interface.is_null_session(ctx.session):
            self.save_session(ctx.session, response)

#ReuqestContext
class ReuqestContext():
    def push(self, object):
        ...
        self.session = self.app.open_session(self.reuqest)
        if self.session is None:
            self.session = self.app.make_null_session()
        ...
```

`session` 是 `RequestContext` 中属性，所以代理说明如下：

```python
session = LocalProxy(partial(_lookup_req_object,'session')
```

### current_app & g

一般来讲， 在 Flask Web 开发时， Flask的实例是延迟创建的。也就是说 `AppContext`还没有压入 `_app_ctx_stack` 中，所以我们在编写代码时，是无法获取完整的 Flask 实例的属性。而当用户访问时，程序的实例已经初始化完成了，因此我们采用 `current_app`代理获取当前 `app`。这仅仅是我的个人理解。实际上这是解决 [多个 Flask 实例运行的问题](http://flask.pocoo.org/docs/0.10/appcontext/)。

`current_app`是获取 `_app_ctx_stack` 栈顶 `AppContext`实例元素的代理.

```python
def _find_app():
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return top.app
current_app = LocalProxy(_find_app)
```

`flask.g` 是存储一下资源信息的，如数据库连接信息。更多应用的则是体现在 Flask 扩展当中。

```python
def _lookup_app_object(name):
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
        return getattr(top,name)
g = LocalProxy(partical(_lookup_app_object, 'g'))

# flask.app.py
class Flask(_PackageBoundObject):
    app_ctx_globals_class = _AppCtxGlobals #实现的是类似字典的功能

# AppContext
class AppContext(object):
    def __init__(self, app):
        self.g = self.app.app_ctx_globals_class()

#RequestContext
class RequestContext(object):
    #定义与request相关的 g 变量
    def _get_g(self):
        return _app_ctx_stack.top.g
    def _set_g(self, value):
        _app_ctx_stack.top.g = value
    g = property(_get_g, _set_g)
    del _get_g, _set_g   
```

上述代码存在一个疑问是 `g` 对象是基于请求的，每次请求都会重置。那么 `g`  为什么不是 `RequestContext` 而是 `AppContext` ? 
[flask.g API 文档](http://flask.pocoo.org/docs/0.10/api/#flask.g) 中说明了 `g` 变量的改动。

---











