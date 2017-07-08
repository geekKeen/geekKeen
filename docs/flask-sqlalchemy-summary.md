# []Flask-SQLAlchemy 学习总结

## 初始化和配置

`ORM`(Object Relational Mapper) 对象关系映射。指将面对对象得方法映射到数据库中的关系对象中。
[Flask-SQLAlchemy](http://flask-sqlalchemy.pocoo.org/2.1/)是一个Flask扩展，能够支持多种数据库后台，我们可以不需要关心SQL的处理细节，操作数据库，一个基本关系对应一个类，而一个实体对应类的实例对象，通过调用方法操作数据库。`Flask-SQLAlchemy`有很完善的文档。

`Flask-SQLAlchemy`是通过URL指定数据库的连接信息的。
初始化的两种方法如下（以连接Mysql数据库为例）：
```python
from flask_sqlalchemy import SQLAlchemy
from flask import FLask
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = \
    "mysql://root:12345@localhost/test"
db = SQLAlchemy(app)
```
或者
```python
from flask_sqlalchemy import SQLAlchemy
from flask import FLask
db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    db.init_app(app)
    return app
```
两者的区别在于:第一种不需要启动flask的`app_context`；但是第二种方法则需要，因为可能会创建多个Flask应用，但是我的理解是一般地开发时，Flask实例是延迟创建的，因为在运行时难以修改配置信息，这种方法符合这种情况。
`Flask-SQLAlchemy`的则需要在`Flask.config`中声明。更多详细信息需要查[配置](http://flask-sqlalchemy.pocoo.org/2.1/config)。例如配置信息中指出`SQLAlchemy`是可以绑定多个数据库引擎。再例如：在新浪SAE云平台开发个人博客时遇到`gone away`这种问题就需要添加`SQLALCHEMY_POOL_RECYCLE`信息,新浪开发者文档中有说明。

---

## SQLALchemy处理 对象->关系
SQLAlchemy是如何处理对象到关系的？实例来自于数据库系统概论内容。
### 简单实例
创建学生students表
```python
class Student(db.Model):
    __tablename__ = 'students' #指定表名
    sno = db.Column(db.String(10), primary_key=True)
    sname = db.Column(db.String(10))
    sage = db.Column(db.Integer)
```
API文档说明创建对象需要继承`db.Model类`关联数据表项，`db.Model类`继承`Query类`提供有数据查询方法；`__tablename__`指定数据库的表名，在`Flask-SQLAlchemy`中是可省的。`Column`指定表字段。

**SQLAlchemy支持字段类型有：**

|类型名|python类型|说明|
|:---:|:---:|:---|
|Integer|int|普通整数，32位|
|Float|float|浮点数|
|String|str|变长字符串|
|Text|str|变长字符串，对较长字符串做了优化|
|Boolean|bool|布尔值|
|PickleType|任何python对象|自动使用Pickle序列化|

来源于[Simple Example](http://flask-sqlalchemy.pocoo.org/2.1/models/#simple-example)，[Flask Web开发](https://segmentfault.com/a/1190000002362175)有更详细的内容。
其余的参数指定属性的配置选项，常用的配置选项如下：

|选项名|说明|
|:---:|:---:|
|primarykey|如果设为True，表示主键|
|unique|如果设为True，这列不重复|
|index|如果设为True，创建索引，提升查询效率|
|nullable|如果设为True，允许空值|
|default|为这列定义默认值|


如使用`default`默认time属性如下:
```python
time = db.Column(db.Date, default=datetime.utcnow)
```
说明`default`可以接受`lambda`表达式。

### 一对多
按创建单张表的方法，创建学院Deptment表
```python
class Deptment(db.Model):
    __tablename__ = 'deptments'
    dno = db.Column(db.Integer, primary_key=True)
    dname = Sname = db.Column(db.String(10),index=True)
```
学院和学生是一对多的关系。`Flask-SQLAlchemy`是通过`db.relationship()`解决一对多的关系。在Dept中添加属性，代码如下：
```python
class Deptment(db.Model):
    ...
    students = db.relationship('Student', backref='dept')
    
    
class Student(db.Model):
    ...
    dept_no = db.Column(db.Integer, db.ForeignKey('deptments.dno'))
```
表的外键由`db.ForeignKey`指定，传入的参数是表的字段。`db.relations`它声明的属性不作为表字段，第一个参数是关联类的名字，`backref`是一个反向身份的代理,相当于在Student类中添加了dept的属性。例如，有Deptment实例dept和Student实例stu。`dept.students.count()`将会返回学院学生人数;`stu.dept.first()`将会返回学生的学院信息的Deptment类实例。一般来讲`db.relationship()`会放在`一`这一边。

### 多对多
多对多的关系可以分解成一对多关系，例如：学生选课，学生与课程之间的关系：
```python
sc = db.Table('sc',
    db.Column('sno', db.String(10), db.ForeignKey('students.sno'))
    db.Column('cno',db.String(10), db.ForeignKey('courses.cno'))
    )
    
Class Course(db.Model):
    __tablename__ = 'courses'
    cno = db.Column(db.String(10), primary_key=True)
    cname = db.Column(db.String(10), index=True)
    students = db.relationship('Student',
         secondary=sc,
         backref=db.backref('course',lazy='dynamic'),
         lazy='dynamic'
         )
```
`sc`表由`db.Table`声明，我们不需要关心这张表，因为这张表将会由`SQLAlchemy`接管，它唯一的作用是作为students表和courses表关联表，所以必须在`db.relationship()`中指出`sencondary`关联表参数。`lazy`是指查询时的惰性求值的方式，[这里](http://flask-sqlalchemy.pocoo.org/2.1/models/#one-to-many-relationships)有详细的参数说明，而`db.backref`是声明反向身份代理，其中的`lazy`参数是指明反向查询的惰性求值方式，`SQLAlchemy`鼓励这种方式声明多对多的关系。

但是如果关联表中有自定义的字段，如sc表中添加成绩字段则需要更改表声明方式,将`sc`更改为继承`db.Model`的对象并设置`sc：courses = 1：n` 和`sc：student = 1：n`的关系。

---

## SQLALchemy处理 关系->对象
[Flask-SQLAlchemy查询](http://flask-sqlalchemy.pocoo.org/2.1/queries/)中有详细的说明。创建关系后该如何查询到对象？

**SQLAlchemy有查询过滤器如下：**

|过滤器|说明|
|:---:|:---|
|filter()|把过滤器添加到原查询，返回新查询|
|filter_by()|把等值过滤器添加到原查询，返回新查询|
|limit()|使用指定值限制原查询返回的结果数量，返回新查询|
|offset()|偏移原查询返回的结果，返回新查询|
|order_by()|排序返回结果，返回新查询|
|groupby()|原查询分组，返回新查询|

这些过滤器返回的结果都是一个新查询，我的理解是这些查询其实是生成的SQL语句，`lazy`的惰性求值方式也体现在查询上，而这些语句不能生成需要查询的对象,需要调用其他的方法生成对象。

**SQL查询执行函数：**

|方法|说明|
|:---:|:---|
|all()|以列表形式返回结果|
|first()|返回第一个结果，如果没有返回None|
|first_or_404()|返回第一个结果，如果没有抛出404异常|
|get()|返回主键对应记录，没有则返回None|
|get_or_404()|返回主键对应记录，如果没有抛出404异常|
|count()|返回查询结果数量|
|paginate()|返回paginate对象，此对象用于分页|





