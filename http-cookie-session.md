#  阅读 RFC 2109 理解 Session 和 Cookie

## 什么是 Session？ 什么是 Cookie？
众所周知， HTTP 是无状态协议，本次请求与下一次请求无关，但为了实现会话机制，实现一条
逻辑上的通信链路机制，则产生了 Session 和 Cookie。

RFC 2109 Session 定义：

> the technique allows clients and servers that
wish to exchange state information to place HTTP requests and
responses within a larger context, which we term a "session".

所以 session 是一系列交换信息的过程， 我们所说的 Cookie 则是需要交换的信息。

一次完整的通话中有几个重要的属性。

1. 通话既有开始，也有结束；
2. 通话是短暂的；
3. 通话的双方都可以终止对话；
4. 通话中交换的状态信息是隐式的；

HTTP 的状态信息转换是如何做的？ 又是怎样对用户透明的？

## 交换 Cookie 的过程 

先介绍交换的过程中需要用到的术语：

    1.`user agent`: 代表用户端的 App；
    2.`origin server`: 服务端服务器，存储有服务的资源;

HTTP 这个逻辑上的会话是由 Server 端创建的。过程如下：

    1. origin server 给 user agent 一个带有 `Set-Cookie` 头的响应；
    2. user-agent 解析 `Set-Cookie` 头，再决定是否建立连接，如果同意，则将信息写入带有`Cookie` 字段的请求头里，发送至服务端。
    3. origin server 收到 Cookie 之后，可以判断是否符合要求，决定是否终止会话， 如果终止会话则在 `Set-Cookie` 中写入 `Max-Age=0` 的信息， user agent 收到后，会将此 Cookie 丢弃

## 状态信息存储
在内容上，RFC 2019 中并没有规定 Cookie 字段和 Set-Cookie 字段是否应该加密，也没有说明以何种加密方式加密。
在实现上，没有强制规定状态信息应该存储在origin server 还是 user agent 上。尤其在 user agent 端，
没有添加任何固定的限制，只要满足一些基本的存储要求和更新算法。

一般有`client side` 和 `Server side` 两种实现方式.

### Client Side Session
Client Side Session 将状态信息存在 user agent 端， 由客户端管理，服务端验证和读取， 
Flask 的默认的实现方式。

但是这种方式存在以下一些缺点：

    1. 当涉及一些用户隐私行为时， 安全性得不到保障， 比如将用户的用户名密码存在客户端；
    2. user agent 的存储能力有限， 在会话过程中可能会存在 cookie 丢失问题；
    3. Cookie 中的数据将会很多；

当然这种方式有一些其他的优点，最典型的应用是，防止重复提交，将提交信息放在 Cookie 中，
Server 判断提交的信息是否已经提交过。 
    
### Sever Side Session

Sever Side Session 将信息存在服务端，一般存在 DB 中。 为了 Sever 和 Client 的认证， Sever 会给 Client 
一个 DB 记录的主键，用于索引信息，这就是常说的 Session-id, 通过 session-id 可以验证 client 是否是已经验证过的

RFC 中说明了 origin server 需要解决 session-id 的碰撞问题，以及 key 的垃圾回收机制。


## 利用 HTTP 协议会话管理的漏洞

HTTP 协议仅通过Cookie验证user agent， 并不能说明这个请求是由用户请求。

可以通过欺骗 user agent 请求双方已经信任过的网站，实现攻击，

CSRF(Cross-site Request forgery)漏洞就是利用这个缺陷。

另外，只要能够获取 user agent 上的认证信息(Cookie)， 
即使是非正常用户，也能正常的访问网站， 因为 Server 只认证信息，没有办法认证这个信息是从哪里来的。
XSS 攻击方式就是利用了这一点。

一些爬虫工具也是利用了这一点，才能够模拟登录。


## 推荐阅读

1.[RFC 2109](https://www.ietf.org/rfc/rfc2109.txt)

2.[Client Side vs Server Side Session](http://phillbarber.blogspot.com/2014/02/client-side-vs-server-side-session.html)

3.[CSRF 跨站请求伪造](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0)

4.[Flask session 实现 flask/sessions.py](https://github.com/pallets/flask/blob/master/flask/sessions.py)