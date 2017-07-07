# Flask 简介


## Web框架
Web框架是构建Web应用的一种方式。尽管现在很多语言如PHP、Java都能开发Web应用，这些语言也都有相应成熟的Web框架，但是请求处理是这些Web框架的核心。[知乎问答-如何学习Web框架](http://www.zhihu.com/question/36625971/answer/68995224) 提到Web框架涉及的基本元素，请求处理是学习Web框架的首要问题。

---

##Flask框架

>[Flask](https://dormousehole.readthedocs.org/en/latest/) 是一个用于 Python 的微型网络开发框架。

Flask的‘微’体现在它只提供Web服务的基本功能，其他的功能是由Flask的扩展实现，用户可以根据需求应用核心扩展。它的基本功能依赖于符合 `WSGI` 规范（Web Server Gateway Interface）的 `Werkzeug` 库和模板系统 `Jinja2`。
当我们通过URL访问网站时，是向Web服务器发送了请求。服务器会根据URL将请求交给相应的Web程序处理。所以服务器与Web应用程序的交互需要一定的规则。而Python专用的规范是WSGI [[PEP-3333](https://www.python.org/dev/peps/pep-3333/)定义]，文章 [WSGI简介](https://segmentfault.com/a/1190000003069785) 给出了简单说明。

---

##Flask核心功能

Flask涉及到两个重要的类——Flask和Blueprint[蓝本] 类。
flask的应用程序需要Flask类实例化才能运行，网站的基本配置信息也包含在此类中。
实例化如下：

```python
from flask import Flask
app = Flask(__name__)#__name__程序的文件名，通过此变量定位资源文件位置
```
配置如下：
```python
app.config['CONFIGURATION'] = "CONFIGURATION"
```
Blueprint类与Flask类似，它能够更好的组织Web应用程序，并能延迟Flask类实例的创建。

###1.路由和请求处理
`路由`的存在是为了将Web服务的请求转交给Flask程序实例的函数处理，简单的说处理URL和函数之间的关系称为路由。函数称为`视图函数`。
`Flask类`包含了route装饰器，通过初始化Flask类注册视图函数。

####无参数
```python
@app.route('/')
def index():
    return "<h1>Hellow World</h1>"
```
当访问网站根域名时，会执行index函数，返回值的结果会在网页中显示。。

####带参数
在很多时候用户不同，使用的URL不同，带参数的路由能够很好的处理URL中变化的部分。如在网页中显示个人名字
```python
@app.route('/user/<name>')
def user(name):
    return '<h1>Hello, {!r}</h1>'.format(name)
```
`<name>`部分是可变部分，name将作为参数传递给视图函数。此部分可以指定name 的类型，如`@app.route('/user/<int:age>')`可指定参数类型
可指定的类型有int，float，path（路径标识）

####处理GET, POST请求
```python
@app.route('/', methods=['GET', 'POST'])
def index():
    pass
```
当网页需要处理表单等请求时，需要添加methods，使得视图函数能够处理http请求。methods中的参数包括了Http协议中定义的5种动作。

####响应处理
```python
from flask import make_request
@app.route('/'):
    response = make_request("<h1>Cookie</h1>")
    response.set_cookie('answer','42')
    return response
```
响应处理中很重要的一部分是处理Http协议的状态码。Flask默认状态码是200。而返回特殊状态码可在返回值中添加，代码如下：
```python
@app.route('/')
def index():
    return "<h1>Bad Request</h1>", 400 #返回特殊状态码
```
####“'?' + 键值对“ 形式
`?` 后的键值对由flask提供request对象处理。假设URL为`127.0.0.1：5000?page=1`
```python
from flask import request

@app.route('/', methods=['GET'])
def index():
    page = request.args.get('page', type=int)
```

###2.异常处理
在浏览网页时，可能会遇到一些错误，flask提供abort函数抛出错误，而抛出的异常会直接返回给Web服务器。我们也可以自定义处理这些异常。代码如下：
```python
from flask import abort
@app.route('/')
def index():
    abort(404) #抛出404异常
    
@app.errorhandler(404) 
def page_not_found(error): # 自定义处理异常
    return 'This page does not exist', 404
```

---

###3.请求钩子

`请求钩子`是指在请求之前或之后所做的处理函数

| 函数名 | 功能 |
| :-------:|:------:|
|before_first_request|在处理第一个请求之前执行|
|before_request|每次请求之前执行|
|after_request|无异常，每次请求之后执行|
|teardown_request|即使异常，也在请求之后执行|

请求钩子函数与视图函数之间通过g变量共享数据。

---

##问题
**Q1：如何调试Web应用程序？**
**Q2：如何评价Web应用程序的性能？**
**Q3：Web应用程序如何做单元测试？**


---

##学习资料
1. [《Flask Web 开发》SegmentFault系列文章](https://segmentfault.com/a/1190000000723218) 作者只录入了前七章内容，未涉及实例开发部分。

2. [《Flask Web 开发》中文版](http://book.douban.com/subject/26274202/)。建议入手第二版。

3. [《Flask Web 开发》作者Blog](http://blog.miguelgrinberg.com/)。书中有任何疑问可以在文章中搜索。

4. [Flask大型教程项目Blog ](http://www.pythondoc.com/flask-mega-tutorial/index.html)。

5. [Flask API文档](https://dormousehole.readthedocs.org/en/latest/api.html?highlight=api)

---






