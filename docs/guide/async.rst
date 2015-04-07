异步非阻塞I/O
---------------------------------

实时的web功能需要为每一个用户保持一个长期存活的，大部分时候空闲的链接。在一个传统的同步web服务器端，这暗示着要为每一个用户奉献出一个线程，这将会带来很大的代价。

为了最大限度地减少并发连接的成本，Tornado使用单线程事件循环。这意味着，所有的应用程序代码应力求异步非阻塞的，因为在一个时间只有一个操作可是活动的。

异步非阻塞的定义密切相关，而且经常互换使用，但它们并不完全是一回事。

阻塞
~~~~~~~~

一个函数在return之前，等待其他事情发生（其他代码执行）的过程，称其为 **阻塞** 状态。一个函数可能会因为很多原因而阻塞：网络I/O, 磁盘I/O, 以及锁等等。事实上，*每一个* 函数运行中并使用CPU的时候至少都会发生一点点阻塞现象（为了说明相对于其他种类的阻塞，为什么需要将CPU的阻塞进行认真对待，这里举一个极端的例子，如 `bcrypt <http://bcrypt.sourceforge.net/>`_ 这样的密码hash函数,会使用几百毫秒的CPU时间，远远超过典型的网络或硬盘访问时延。

一个函数可能在某些方面阻塞，而在其他方面却并不阻塞。比如，`tornado.httpclient` 类在默认的配置情况下，会在DNS解析上阻塞，而不会再其他的网络访问上阻塞（ 可以使用 `.ThreadedResolver` 或者 基于正确配置的 ``libcurl`` 构建环境下的 ``tornado.curl_httpclient`` 类来减轻阻塞现象）。在Tornado中，我们通常讨论的是网络I/O上的阻塞，虽然各种情况的阻塞都应该被最小化。

异步
~~~~~~~~~~~~

一个 **异步** 的函数会在它执行完成之后就先返回（Return），并且在触发一些即将发生的应用程序中的行为之前，通常会在后台进行一系列的工作（而对于正常的 **同步** 函数来说，任何工作都是在程序返回之前完成的）。 下面列举了几种不同的异步接口的开发方式：

* 回调参数（Callback argument）
* 返回一个占位符 (`.Future`, ``Promise``, ``Deferred``)
* 传送到队列
* 回调注册表 (比如 POSIX 信号)

无论使用哪种的接口开发方式， *顾名思义* 异步函数都需要与调用方有不同的交互；不可能在对调用方透明的情况下使一个同步的函数异步化。（ `gevent
<http://www.gevent.org>`_ 系统使用轻量级的线程模式，能够达到一个可以与异步系统媲美的性能，但它们实际上并没有进行任何异步化。）

例子
~~~~~~~~

下面是一个同步函数的样例：

.. testcode::

    from tornado.httpclient import HTTPClient

    def synchronous_fetch(url):
        http_client = HTTPClient()
        response = http_client.fetch(url)
        return response.body

.. testoutput::
   :hide:

而下面是一个用回调参数模式重写的异步函数的样例：

.. testcode::

    # 使用了内置的异步HTTP客户端
    from tornado.httpclient import AsyncHTTPClient 

    # 回调参数模式
    def asynchronous_fetch(url, callback):
        http_client = AsyncHTTPClient()
        def handle_response(response):
            callback(response.body)
        http_client.fetch(url, callback=handle_response)

.. testoutput::
   :hide:

再下面是使用占位符 `.Future` 方式替代回调方式的样例：

.. testcode::

    from tornado.concurrent import Future

    def async_fetch_future(url):
        http_client = AsyncHTTPClient()
        my_future = Future()
        fetch_future = http_client.fetch(url)
        fetch_future.add_done_callback(
            lambda f: my_future.set_result(f.result()))
        return my_future

.. testoutput::
   :hide:

原始的 `.Future` 版本更为复杂，但是在Tornado中，使用 ``Futures`` 仍然是被推荐的做法，主要因为它有两个很重要的优势：错误处理方式不需要改变，你可以在 `.Future.result` 方法中简单的抛出异常（而回调方式的接口开发需要有特殊的错误处理方式），并且 ``Futures`` 可以很方便的和协程配合使用。 协程将会在下一章节中深入讨论。 上述示例使用协程进行配合后，和原生的同步版本的函数看起来十分类似：（不过这里引入了yield迭代）

.. testcode::

    from tornado import gen

    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        raise gen.Return(response.body)

.. testoutput::
   :hide:

``raise gen.Return(response.body)`` 语法在Python 2(以及3.2)版本中是神器，在这些版本中生成器是不允许有返回值的。 为了克服这个问题，Tornado的协程抛出了一个被成为 `.Return` 的特殊种类的异常类。协程捕获这种异常，并且将它以返回值的方式处理。在Python 3.3以及更高版本中，可以直接通过 ``return
response.body`` 的方式达到同样的效果（Python 3.3以及更高版本中支持在生成器中存在返回值）。

