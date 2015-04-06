介绍
------------

`Tornado <http://www.tornadoweb.org>`_ 是一个最初由 `FriendFeed
<http://friendfeed.com>`_ 团队开发的Python web框架及异步网络类库。依托于非阻塞的网络I/O，Tronado可以支持数以万记的活跃链接，十分适用于如 `long polling <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_,
`WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_, 以及其他需要每个用户都保持一个活跃长连接的应用。


Tornado 大致可以分为四个主要部件:

* 一个web框架（包括 `.RequestHandler` 子类，用于创建Web应用程序，以及各种支持类 ）
* Client- and server-side implementions of HTTP (`.HTTPServer` and
  `.AsyncHTTPClient`).
* HTTP的客户端和服务器端的实现（`.HTTPServer` 和 `.AsyncHTTPClient` ）
* 一个异步的网络库（`.IOLoop` 和 `.IOStream`），作为HTTP组件的一部分，也可以用于实现其他的协议。
* 一个协程库 (`tornado.gen`)，相比回调方式，该类库可以用更简单的方式进行异步的代码开发。


Tornado的web框架和HTTP服务器共同提供了一个全栈式的可用于替代 `WSGI <http://www.python.org/dev/peps/pep-3333/>`_ 的开发框架。然而，也可以直接将Tornado的web框架运行在WSGI容器 (`.WSGIAdapter`) 之中，或者将Tornado的HTTP服务器作为其他的WSGI框架的容器 (`.WSGIContainer`)，这两种组合都有自己的局限性，为了更充分的利用Tornado的功能，最好将Tornado的web框架和HTTP服务器一起使用。
