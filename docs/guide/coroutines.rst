协程
==========

.. testsetup::

   from tornado import gen

在Tornado中 推荐使用 **协程** 进行异步代码的开发。协程使用Python的 ``yield`` 关键字将程序挂起和恢复执行，从而代替使用一连串的回调方式（ 像在 `gevent <http://www.gevent.org>`_ 中出现的轻量级线程合作的方式有时也被成为协程，但是在Tornado中所有的协程使用显式上下文切换，并且被称为异步函数。



使用协程开发是最接近于同步开发的方式，并且不用浪费额外的线程。 并且通过减少上下文转换的频率，协程 `使并发编程变得更容易 <https://glyph.twistedmatrix.com/2014/02/unyielding.html>`_ 。

示例::

    # 引入协程库gen
    from tornado import gen

    # 使用gen.coroutine修饰器
    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        # 在Python 3.3版本之前，在生成器中是不允许有返回值的
        #（既不允许在有生成器的函数内，使用return语句，Python解释器会报错）
        # 所以，必须通过使用抛出异常的方式代替
        # 如使用 raise gen.Return(response.body）
        return response.body

协程是如何工作的？
~~~~~~~~~~~~~~~~

A function containing ``yield`` is a **generator**.  All generators
are asynchronous; when called they return a generator object instead
of running to completion.  The ``@gen.coroutine`` decorator
communicates with the generator via the ``yield`` expressions, and
with the coroutine's caller by returning a `.Future`.

Python中包含关键字 ``yield`` 的函数被称为生成器。所有的生成器都是异步的；当调用生成器的时候，会返回一个生成器的对象，而不是一直运行到结束。``@gen.coroutine`` 修饰器通过 ``yield`` 表达式与生成器进行交流，而且通过返回一个 `.Future` 与协程的调用方进行交互。

下面是一个协程修饰器内部循环的简单版本示例::

    # tornado.gen.Runner 类的简单内部循环
    def run(self):
        # send(x) makes the current yield return x.
        # It returns when the next yield is reached
        future = self.gen.send(self.next)
        def callback(f):
            self.next = f.result()
            self.run()
        future.add_done_callback(callback)

修饰器从生成器中接受到一个 `.Future` 对象 ，然后等待（非阻塞的）这个 `.Future` 对象 执行完成，然后“解开”这个 `.Future` 对象并且将结果作为 ``yield`` 表达式的结果传回给生成器。 除了直接通过异步函数将这个 `.Future` 对象回传给一个 ``yield`` 表达式以外，大多数的异步代码都不会接触到 这个 `.Future` 对象 。

协程模式
~~~~~~~~~~~~~~~~~~

与回调的交互
^^^^^^^^^^^^^^^^^^^^^^^^^^

To interact with asynchronous code that uses callbacks instead of
`.Future`, wrap the call in a `.Task`.  This will add the callback
argument for you and return a `.Future` which you can yield:

代替 `.Future` 使用回调的方式与异步代码进行交互，将调用封装在一个 `.Task` 里。

.. testcode::

    @gen.coroutine
    def call_task():
        # Note that there are no parens on some_function.
        # This will be translated by Task into
        #   some_function(other_args, callback=callback)
        yield gen.Task(some_function, other_args)

.. testoutput::
   :hide:

调用阻塞的函数
^^^^^^^^^^^^^^^^^^^^^^^^^^

The simplest way to call a blocking function from a coroutine is to
use a `~concurrent.futures.ThreadPoolExecutor`, which returns
``Futures`` that are compatible with coroutines::

    thread_pool = ThreadPoolExecutor(4)

    @gen.coroutine
    def call_blocking():
        yield thread_pool.submit(blocking_func, args)

Parallelism
^^^^^^^^^^^

The coroutine decorator recognizes lists and dicts whose values are
``Futures``, and waits for all of those ``Futures`` in parallel:

.. testcode::

    @gen.coroutine
    def parallel_fetch(url1, url2):
        resp1, resp2 = yield [http_client.fetch(url1),
                              http_client.fetch(url2)]

    @gen.coroutine
    def parallel_fetch_many(urls):
        responses = yield [http_client.fetch(url) for url in urls]
        # responses is a list of HTTPResponses in the same order

    @gen.coroutine
    def parallel_fetch_dict(urls):
        responses = yield {url: http_client.fetch(url)
                            for url in urls}
        # responses is a dict {url: HTTPResponse}

.. testoutput::
   :hide:

Interleaving
^^^^^^^^^^^^

Sometimes it is useful to save a `.Future` instead of yielding it
immediately, so you can start another operation before waiting:

.. testcode::

    @gen.coroutine
    def get(self):
        fetch_future = self.fetch_next_chunk()
        while True:
            chunk = yield fetch_future
            if chunk is None: break
            self.write(chunk)
            fetch_future = self.fetch_next_chunk()
            yield self.flush()

.. testoutput::
   :hide:

Looping
^^^^^^^

Looping is tricky with coroutines since there is no way in Python
to ``yield`` on every iteration of a ``for`` or ``while`` loop and
capture the result of the yield.  Instead, you'll need to separate
the loop condition from accessing the results, as in this example
from `Motor <http://motor.readthedocs.org/en/stable/>`_::

    import motor
    db = motor.MotorClient().test

    @gen.coroutine
    def loop_example(collection):
        cursor = db.collection.find()
        while (yield cursor.fetch_next):
            doc = cursor.next_object()
