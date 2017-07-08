# 深入 Tornado 源码了解 IOLoop 和 异步协程


---


## Tornado 的异步协程说明

Tornado 特性包括了异步的网络库以及协程, 异步网络库的实现依赖于 IO 多路复用 select/ epoll, 通过监听可用连接处理事件; 协程通过实现 concurrent 中的 Future 模式兼容基于回调,基于 yield 生成器以及 Python3 中 await/async.

## IOLoop 事件循环
Tornado 采用在单线程内采用事件循环的方式的减少并发连接的消耗,通过采用 select/epoll/ kqueue 监听可用 fd 处理事件. Loop循环原理的代码摘要如下:

```python
class Configurable(object):
    def __new__(cls, *args, **kwargs):
        # 工厂模式, 实现类实例根据所需配置创建
        impl = cls.configured_class()
        instance = super(Configureable, cls).__new__(impl)
        instance.initalize(*args, *kwargs)
        return instance

class IOLoop(Configurable):
    @classmethod
    def configured_class(cls):
        # 根据操作系统返回相应的 EventLoop 类
        ...
    
class PollLoop(IOLoop):
    # 事件循环的操作在此定义
    # tornado.plateform 中 SelectIOLoop 等都继承此类
    # 原理类似生产者/消费者模式
    # 生产者添加事件/消费者处理事件
    def initalize(self, impl, *args, **kwargs):
        self._impl = impl 
        ...
    
    def add_handler(self, fd, handler, event):
        # 生产者方法
        # 向 Loop 中添加事件
        fd, obj = self.split_fd(fd)
        ...
        self._impl.register(fd, event)
        
    def start(self):
        # 消费者方法
        # 通过轮询事件,处理可用事件
        ...
        while True:
            ...
            event_pairs = self._impl.poll(poll_timeout)
            # 处理事件
```

PollLoop 中 _impl 属性是什么? 如何监听 fd 上发生的事件? 
### select/ epoll 
select / epoll 是 IO 复用模式中的系统调用. select 是通过轮询的方式检测文件 fd 是否可用; epoll 是 select 的改进版本, epoll 与 select 相比,性能优但是兼容性差. Tornado 通过操作系统选择使用某个 IO 复用的系统调用.

### 事件循环 eventloop 单例模式
Tornado 每个线程中只有一个 IOLoop 实例, 在单线程中向 Loop 中注册事件,当事件可用时处理事件. Tornado 通过单例模式,实现每个线程中仅有一个 IOLoop

```python
class IOLoop(Configurable):
    _instance_lock = threading.lock()
    _current = threading.local()
    
    @staticmethod
    def instance():
        # 返回一个全局 Loop 实例 
        if not hasattr(IOLoop, '_instance'):
            with IOLoop._instance_lock:
                if not hasattr(IOLoop, '_instance'):
                　　＃　先检查线程锁，如果未锁检查,当前线程IOLoop是否有Instance
                    IOLoop._instance = IOLoop()
        return IOLoop._instance
    
    def make_current(self):
        # 设置当前线程的 IOLoop 实例为self (IOLoop 实例)
        IOLoop._current.instance = self
        
    @staicmethod
    def current(instance=True):
        # 获取当前线程实例
        current = getattr(IOLoop._current, 'instance', None)
        if current is None and instance:
            return IOLoop.instance()
        return current  
```

通过单例模式, 使得 IOLoop 实例是一个全局变量, 全局都向同一个IOLoop中注册事件.


## concurrent Future 模式
`同步` 是值事件按照一定次序进行,而 `异步` 表示事件之间没有严格的时间关系.
`Future` 模式是指在函数中,用 Future 实例代替返回值, Future 实例中保存当前函数的状态, 此状态包括要返回的结果,或者抛出的异常. 然后在事件循环中恢复这些状态,将结果返回给调用者. 因此函数执行的结果与执行过程非同步发生,这就是异步. 

异步实现的过程包括两个问题: 
1. 状态如何能保存
2. 这些状态又如何恢复,并持续执行.

### Future 保存运行状态

```python
class Future(object):
    def __init__(self):
        self._result = None
        self._exc_info = None
        self._callbacks = []
        self.running = True
        
    def set_result(self, result):
        ...
        
    def set_exc_info(self, exce_info):
        ...
        
    def result(self):
        ...
    
    def exc_info(self):
        ...
        
    def add_done_callback(self, calback):
        self._callbacks.append(callback)
``` 

`Future` 作为返回结果的占位符需要保存运行状态,且能够恢复状态. 这里的 `callback` 就是恢复运行状态,返回实际结果的回调函数.这个回调函数的参数必须是 future 实例. 如果异步操作发生异常,则将异常信息保存在 Future 当中. 一般使用 `sys.exce_info()` 做传递参数.

### 异步回调的调度

异步操作一般用于 IO 资源调度时, 而对于计算密集的操作,异步操作并不试用. 这些调度则依赖于 IOLoop 的事件循环, 通过系统调用,找到可用资源. `_make_coroutines_wrapper` 和 `Runner` 中调度的核心. `gen.engine` 和 `gen.coroutine` 实现依赖与 `_make_coroutines_wrapper`. Runner 则是调度的主要, 它运行可用的 future, 并将暂不可用的 future 注册在 事件循环中. 

关于 Runner 的代码. `run` 和 `handle_yield` 的实现并不清晰, 主要体现在 runner 的循环中调用 handle yield 改变了 self.future 的值,使得 future 能够继续迭代下去. 另外 `_make_coroutines_wrapper` 中存在一些关于 python GC 机制的引用. 如 `_future_to_runner` , 当然异常的捕获也是很重要的.






