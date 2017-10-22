# 记使用类装饰器产生的 Bug，闭包的特性

## 为什么会使用类装饰器？
在最近的开发任务中，需要为 `Model` 添加触发器，开发过程中发现这些触发器的动作和内容几乎相同，
因此想将这些操作全部都提取出来做成类的装饰器。

```python
from sqlalchemy.dialects.postgresql import ARRAY
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Model(db.Model):
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    attr1 = db.Column(ARRAY(db.Integer, dimensions=1))
    merged_attr1 = db.Column(ARRAY(db.Integer, dimensions=1))
    attr2 = db.Column(ARRAY(db.DateTime, dimensions=1))
    merged_attr2 = db.Column(ARRAY(db.DateTime, dimensions=1))
    
    @staticmethod
    def on_changed_attr1(target, value, oldvalue, initiator):
        target.merged_attr1 = list(set(value))
        
    @staticmethod
    def on_changed_attr2(target, value, oldvalue, initiator):
        target.merged_attr2 = list(set(value))
        
db.event.listen(Model.attr1, 'set', Model.on_changed_attr1)
db.event.listen(Model.attr2, 'set', Model.on_changed_attr2)
```

## 尝试写出类装饰器

上一部分的代码很简单，就是利用触发器(事件)机制， 当设置 `attr1` or `attr2` 的值时去重，
并存储，当然其他 Model 可能也会有相同的代码。以上部分几个相同的代码段，如果数量过多，
会人为的编码失误造成错误，另外如果写代码时如果逻辑混乱，数据内容会产生一些错误，不好调试。
我想把他们提取出来，所以写出了如下带 Bug 的装饰器。

```python
def field_set_trigger(fields=None):
    def wrapper(cls):
        for field in fields or ():
            def on_changed(target, value, oldvalue, initiator):  # haha, Bug!
                setattr(target, 'merged_%s' % field, list(set(value)))
            db.event.listen(getattr(cls, field), 'set', on_changed)
        return cls
    return wrapper
    
@field_set_trigger(['attr1', 'attr2'])
class Model(db.Model):
    # ...
    pass
``` 

## 装饰器中存在的 Bug

`field_set_trigger` 是一个类装饰器，为指定的 Model 的字段添加监听器。 我也只是提取出一个
for 的循环，但是 run 的时候出 bug 了， 错误提示大致是类型错误 datetime 类型接到了整型值。
先抛出解决方法：

```python
def field_set_trriger(fields=None):
    def wrapper(cls):
        for field in fields:
            def on_changed(target, value, oldvalue, intiator, field=field):
                # look params carefully
                setattr(target, 'merged_%s' % field, list(set(value)))
            db.event.listen(getattr(cls,field), 'set', on_changed)
    return wrapper
    
@field_set_trriger
class Model(db.Model):
    # ...
    pass
```

仔细看 `on_changed` 函数的参数，添加了 `field` 默认参数，参数值为 `field`，解决了问题。

## Why？ 为什么会产生这个 bug？
如果熟悉 Python 一些常用的坑，应该会了解到 lambda 中有一个坑：

```python
funcs = [lambda x : x*i for i in range(5)] 
for func in funcs:
    print func(2)
# All of output are 8

# fix 
funcs = [lambda x, i=i : x*i for i in range(5)] 
```

这个类装饰器存在的问题就是这个坑。问题的主要原因是 Python 闭包特性：内层的函数可以访问外层的变量，
触发器中的声明和定义是将  `on_changed` 这个可调用对象当参数传递。 每次触发器操作调用时，
找 `field` 值， 而每次找到的值都是装饰器中 `fields` 迭代完成的最后值，所以才会出现类型错误。

Q: 为什么默认参数就能解决问题？
A: 因为提供默认参数后， 每一次 field 都是从 on_changed函数自己的空间中找，而且每次生成的
on_changed函数参数的默认值都是本次迭代的值，所以才能解决问题。

Q：如果这 on_changed 的默认参数是可变类型是否影响结果？
A：Python 中关于默认参数一节有一个问题是默认参数不能为可变类型，

默认参数实例代码：

```python
def default(element, my_list):
    my_list.append(element)
    return my_list
    
l1 = default(1)  # [1]
l2 = default(2)  # [1, 2]
```

这里的原因是因为 default 是可调用对象，对象调用时，dafault 会去找 `my_list`, 因为对象内部
会改变 my_list值，所以才会发现行为与预期不同。注意这里的 default 是一个可调用的函数对象。

这个行为如果应用的上述的类描述中是否也会产生这种影响呢？
答案是不会， 因为 for 循环每次都会产生一个可调用的对象，只是每次产生对象的名字是 on_changed
他们不是同一对象。 

## end， 结尾
以上通过生产过程中写一个类装饰器产生的 bug 来说明一些我们可能镌刻在心头，记得滚瓜烂熟的坑， 
讲述了这些坑的原理以及解决方案的原理。
在文章的最后，我还将一些记忆中闭包的坑记录在这里，以此充分阐述闭包的特性。 

### 闭包中变量 bug

```python
def decorator(func):
    var = 1
    def wrapper(*arg, **kwargs):
        print var  # raise Unbound error
        var = 2
    return wrapper
```

以上代码段一定会出错，我的理解是可能这个 bug 与编译原理的词法解析有一点的关联，Python 解释器
在解析 wrapper 代码时找到 `var = 2`， 将 var 解析为 wrapper 空间下的临时变量。
调用时 wrapper 会去找 var 的值，却发现此时的 var 没有绑定。 

解决这个问题的方法有两种：
1. 在 Py2 中，将 var 声明为全局变量 global；
2. 在 Py3 中，在 wrapper 中 var 定义为 nonlocal；

### 随手记录一个习题

```python
def count():
    fs = []
    for i in range(3):
        fs.append(lambda : i * i)
    return fs
   
f1, f2, f3 = count()

print f1(), f2(), f3()  # result?
```

原理与本文中说明的内容相关。