异步非阻塞I/O
---------------------------------

实时的web功能需要为每一个用户保持一个长期存活的，大部分时候空闲的链接。在一个传统的同步web服务器端，这暗示着要为每一个用户奉献出一个线程，这将会带来很大的代价。

为了最大限度地减少并发连接的成本，Tornado使用单线程事件循环。这意味着，所有的应用程序代码应力求异步非阻塞的，因为在一个时间只有一个操作可是活动的。

异步非阻塞的定义密切相关，而且经常互换使用，但它们并不完全是一回事。

阻塞
~~~~~~~~

一个函数在return之前，等待其他事情发生（其他代码执行）的过程，称其为 **阻塞** 状态。一个函数可能会因为很多原因而阻塞：网络I/O, 磁盘I/O, 以及锁等等。事实上，*每一个* 函数运行中并使用CPU的时候至少都会发生一点点阻塞现象（为了说明相对于其他种类的阻塞，为什么需要将CPU的阻塞进行认真对待，这里举一个极端的例子，如 `bcrypt <http://bcrypt.sourceforge.net/>`_ 这样的密码hash函数,会使用几百毫秒的CPU时间，远远超过典型的网络或硬盘访问时延。

A function can be blocking in some respects and non-blocking in
others.  For example, `tornado.httpclient` in the default
configuration blocks on DNS resolution but not on other network access
(to mitigate this use `.ThreadedResolver` or a
``tornado.curl_httpclient`` with a properly-configured build of
``libcurl``).  In the context of Tornado we generally talk about
blocking in the context of network I/O, although all kinds of blocking
are to be minimized.

Asynchronous
~~~~~~~~~~~~

An **asynchronous** function returns before it is finished, and
generally causes some work to happen in the background before
triggering some future action in the application (as opposed to normal
**synchronous** functions, which do everything they are going to do
before returning).  There are many styles of asynchronous interfaces:

* Callback argument
* Return a placeholder (`.Future`, ``Promise``, ``Deferred``)
* Deliver to a queue
* Callback registry (e.g. POSIX signals)

Regardless of which type of interface is used, asynchronous functions
*by definition* interact differently with their callers; there is no
free way to make a synchronous function asynchronous in a way that is
transparent to its callers (systems like `gevent
<http://www.gevent.org>`_ use lightweight threads to offer performance
comparable to asynchronous systems, but they do not actually make
things asynchronous).

Examples
~~~~~~~~

Here is a sample synchronous function:

.. testcode::

    from tornado.httpclient import HTTPClient

    def synchronous_fetch(url):
        http_client = HTTPClient()
        response = http_client.fetch(url)
        return response.body

.. testoutput::
   :hide:

And here is the same function rewritten to be asynchronous with a
callback argument:

.. testcode::

    from tornado.httpclient import AsyncHTTPClient

    def asynchronous_fetch(url, callback):
        http_client = AsyncHTTPClient()
        def handle_response(response):
            callback(response.body)
        http_client.fetch(url, callback=handle_response)

.. testoutput::
   :hide:

And again with a `.Future` instead of a callback:

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

The raw `.Future` version is more complex, but ``Futures`` are
nonetheless recommended practice in Tornado because they have two
major advantages.  Error handling is more consistent since the
`.Future.result` method can simply raise an exception (as opposed to
the ad-hoc error handling common in callback-oriented interfaces), and
``Futures`` lend themselves well to use with coroutines.  Coroutines
will be discussed in depth in the next section of this guide.  Here is
the coroutine version of our sample function, which is very similar to
the original synchronous version:

.. testcode::

    from tornado import gen

    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        raise gen.Return(response.body)

.. testoutput::
   :hide:

The statement ``raise gen.Return(response.body)`` is an artifact of
Python 2 (and 3.2), in which generators aren't allowed to return
values. To overcome this, Tornado coroutines raise a special kind of
exception called a `.Return`. The coroutine catches this exception and
treats it like a returned value. In Python 3.3 and later, a ``return
response.body`` achieves the same result.
