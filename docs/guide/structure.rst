.. currentmodule:: tornado.web

.. testsetup::

   import tornado.web

Tornado web应用程序架构
======================================

一个Tornado web应用程序通常由一个或一个以上的 `.RequestHandler` 子类组成，一个 `.Application`  对象构成了请求到句柄的路由映射，并且使用一个 ``main()`` 函数来开启服务端。

这里举一个最小的 "hello world" 的例子来说明一下：

.. testcode::

    from tornado.ioloop import IOLoop 
    from tornado.web import RequestHandler, Application, url 

    class HelloHandler(RequestHandler):
        def get(self):
            self.write("Hello, world")

    def make_app():
        return Application([
            url(r"/", HelloHandler),
            ])

    def main():
        app = make_app()
        app.listen(8888)
        IOLoop.current().start()

.. testoutput::
   :hide:

``Application`` 对象
~~~~~~~~~~~~~~~~~~~~~~~~~~

`.Application` 对象负责全局的配置，包括所有请求到句柄的路由映射表。

路由表是由一些 `.URLSpec` 对象组成的列表（或元组），每个对象（至少）包含一个正则表达式和一个句柄类(RequestHandler类的子类)。按照顺序进行匹配，第一个匹配成功的规则将会被使用。如果某个正则表达式包含捕获组，这些捕获组将会成为 *路径参数* 并且会被传送给句柄类的HTTP方法。如果一个字典作为某个 `.URLSpec` 对象的第三个元素，它将会作为 *初始化参数* 传递给 `.RequestHandler.initialize` 方法。最后，`.URLSpec` 对象可以进行命名，并且可以通过 `.RequestHandler.reverse_url` 方法中使用到它。

举个例子：这下面的代码片段中，根地址URL ``/`` 被映射到 ``MainHandler`` 对象，并且映射 ``StoryHandler`` 对象的URL ``/story/`` 中含有一个数字正则。对应的数字会被传给（作为字符串)  ``StoryHandler.get`` 方法。

::

    class MainHandler(RequestHandler):
        def get(self):
            # 通过查找 `.URLSpec` 对象名称story，找到对应的URL，并且传入参数1
            # 最后的URL为 "/story/1"
            self.write('<a href="%s">link to story 1</a>' %
                       self.reverse_url("story", "1")) 


    class StoryHandler(RequestHandler):
        # `.URLSpec` 对象 中传入的第三个字典参数直接传递给initialize方法
        def initialize(self, db):
            self.db = db

        def get(self, story_id):
            self.write("this is story %s" % story_id)

    app = Application([
        url(r"/", MainHandler),
        url(r"/story/([0-9]+)", StoryHandler, dict(db=db), name="story")
        ])


`.Application` 类的构造函数包含了很多的关键字参数，使用这些关键字可以用来定制应用程序的行为以及开启可选功能。点击这里查看 `.Application.settings` 类的完整列表。


``RequestHandler`` 子类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个Tornado的web应用的大部分功能都是在 `.RequestHandler` 子类中完成的。 一个handler子类主要的入口点是一个正在被处理的HTTP方法：``get()``,
``post()``, 等等。每一个handler可以定义多个方法来处理不同的HTTP行为。如上所述，路由表对应该子类的URL的捕获组（正则表达式）将作为参数传入到这些方法之中。

在一个handler之中，可以调用如 `.RequestHandler.render` 或
`.RequestHandler.write` 方法来生成一个response响应。 ``render()`` 方法通过加载一个命名的 `.Template` 对象并通过传入的参数进行渲染。 ``write()`` 方法用来作为不使用模板时的基本输出，支持输入字符串、字节流或者字典（字典将会被编码成JSON格式）。

`.RequestHandler` 类中的许多方法被设计成需要在子类中进行重写并贯彻整个应用。定义一个 ``BaseHandler`` 类并重写
如 `~.RequestHandler.write_error` 和 `~.RequestHandler.get_current_user` 这样的方法，然后使用重写过的``BaseHandler`` 代替 `.RequestHandler` 作为所有自定义handler类的父类。


处理请求的输入
~~~~~~~~~~~~~~~~~~~~~~

The request handler can access the object representing the current
request with ``self.request``.  See the class definition for
`~tornado.httputil.HTTPServerRequest` for a complete list of
attributes.

# TODO: 这句不是很好理解
可以通过访问 ``self.request`` 查看当前对象的请求数据。可以查看 `~tornado.httputil.HTTPServerRequest` 类的定义，了解完整的属性列表。

使用HTML forms形式的请求数据将会被解析，可以通过使用如 `~.RequestHandler.get_query_argument` （get方法使用)
和 `~.RequestHandler.get_body_argument` (post方法使用) 方法获取请求数据。


.. testcode::

    class MyFormHandler(RequestHandler):
        def get(self):
            # 通过get方法将得到一个html form，填写完成后点击submit将调用post方法
            self.write('<html><body><form action="/myform" method="POST">'
                       '<input type="text" name="message">'
                       '<input type="submit" value="Submit">'
                       '</form></body></html>')

        def post(self):
            # post方法中通过self.get_body_argument("message")来取得上面form中post的数据message
            self.set_header("Content-Type", "text/plain") # 设置header
            self.write("You wrote " + self.get_body_argument("message"))

.. testoutput::
   :hide:

由于HTML表单的提交的参数可能是一个单一值，也有可能是一个含有多个元素的列表， `.RequestHandler` 对象提供了一个明确的方法来允许应用程序来判断是否使用单一值还是列表。如果是列表，请使用 `~.RequestHandler.get_query_arguments` 和 `~.RequestHandler.get_body_arguments` ，否则请使用 `~.RequestHandler.get_query_argument` 和 `~.RequestHandler.get_body_argument` 获取单一值。


Files uploaded via a form are available in ``self.request.files``,
which maps names (the name of the HTML ``<input type="file">``
element) to a list of files. Each file is a dictionary of the form
``{"filename":..., "content_type":..., "body":...}``.  The ``files``
object is only present if the files were uploaded with a form wrapper
(i.e. a ``multipart/form-data`` Content-Type); if this format was not used
the raw uploaded data is available in ``self.request.body``.
By default uploaded files are fully buffered in memory; if you need to
handle files that are too large to comfortably keep in memory see the
`.stream_request_body` class decorator.

# TODO: 文件上传之前没有用过，实践之后再确认一些这段的翻译内容
通过表单上传的文件可以通过 ``self.request.files`` 进行访问，文件名（ HTML表单中 ``<input type="file">`` 元素的名字） 映射成一个文件列表。每一个文件是一个像 ``{"filename":..., "content_type":..., "body":...}`` 这样的表单字典。 ``文件`` 对象必须通过包装(即Content-Type使用 ``multipart/form-data``)才有效。如果不适用这种方式包装，原始的上传数据会存在于 ``self.request.body`` 。 默认情况下上传的文件在内存中完全进行缓存。如果你需要处理一些很大的文件，且不希望占用过多的系统内存，可以使用 `.stream_request_body` 类修饰器。

由于HTML表单特别的编码方式（既，可能是单值也可能是多指的歧义问题），Tornado 没有尝试统一不同输入形式的表单参数。特别需要注意的是，Tornado并没有解析JSON格式的请求体(JSON request bodies)。使用JSON格式的应用程序（如REST API）可以重写 `~.RequestHandler.prepare` 方法来解析请求。如下面所示： 

    def prepare(self):
        if self.request.headers["Content-Type"].startswith("application/json"):
            self.json_args = json.loads(self.request.body)
        else:
            self.json_args = None

重写RequestHandler方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了 ``get()``/``post()``等方法，在 `.RequestHandler` 类中还有其他的方法也允许在子类中进行重写。在每一次请求中，下面的一些列调用都会发生：

1. 在每一次请求中，一个新的 `.RequestHandler` 对象都会被创建。
2. `~.RequestHandler.initialize()` 会被优先调用，在 `.Application` 里面配置的字典参数会作为该方法的输入参数。``initialize`` 初始化操作通常只是将输入的参数传递该类对象的成员变量保存，而不应该产生任何的输出和方法调用。
3. 之后 `~.RequestHandler.prepare()` 方法会被调用。 无论哪一个HTTP方法被使用， ``prepare``  都会被调用，它是在所有自定义的子类中共享的最有用的方法。可以使用 ``prepare`` 方法产生输入；如果在该方法中调用了 `~.RequestHandler.finish` （或者 ``redirect`` 等）方法，本次处理就会在这停止。
4. 接下来 ``get()``, ``post()``, ``put()`` 等HTTP方法将会被调用，如果URL的正则表达式包含了捕获组，将会将其作为参数传给对应的方法。
5. 当请求处理完成时， `~.RequestHandler.on_finish()` 方法会被调用。对于同步的请求来说，在 ``get()`` (等)方法返回后会立刻执行；而对于异步请求，它会在调用 `~.RequestHandler.finish()` 方法完成后才执行。

就像在 `.RequestHandler` 文档中记录的那样，所有的方法都是被设计在子类重写的。一些最常用的重写方法包括：

- `~.RequestHandler.write_error`  - 用来在错误页面输出HTML代码。
- `~.RequestHandler.on_connection_close` - 当客户端断开连接的时候会被调用；应用程序可以探测到这种情况并停止之后的处理过程；需要注意的是，一个关闭的连接并不能保证及时的被发现。
- `~.RequestHandler.get_current_user` - 请查看 :ref:`user-authentication`
- `~.RequestHandler.get_user_locale` - 为当前的用户返回 `.Locale` 对象
- `~.RequestHandler.set_default_headers` - 可以用来在返回包中加入额外的headers（如用户自定义的 ``Server`` header）


错误处理
~~~~~~~~~~~~~~

如果处理过程中抛出了一个异常，Tornado将会调用 `.RequestHandler.write_error` 方法来生产一个错误页面。可以使用 `tornado.web.HTTPError` 方法来生成一个特殊状态码的异常；而所有的其他异常都会返回一个HTTP 500 错误。

默认的错误页在debug模式会包含一个stack trace以及一行错误描述（如，"500: Internal Server Error"）。如果想生成自定义的错误页，需要重写 `RequestHandler.write_error` 方法（可以写一个基类用于给其他子类共享）。 这个方法通常可以使用 `~RequestHandler.write` 和 `~RequestHandler.render` 方法产生输出。

# TODO: 这段暂时没理解

If the error was caused by an exception, an ``exc_info`` triple will
be passed as a keyword argument (note that this exception is not
guaranteed to be the current exception in `sys.exc_info`, so
``write_error`` must use e.g.  `traceback.format_exception` instead of
`traceback.format_exc`).

# TODO: 这段理解的也不好

It is also possible to generate an error page from regular handler
methods instead of ``write_error`` by calling
`~.RequestHandler.set_status`, writing a response, and returning.
The special exception `tornado.web.Finish` may be raised to terminate
the handler without calling ``write_error`` in situations where simply
returning is not convenient.

也可以通过调用 `~.RequestHandler.set_status` 设置状态码，写回相应数据，并且返回。用这样的方式来代替 ``write_error`` 方法来生成错误页。在简单的返回不方便的情况下，可以抛出 `tornado.web.Finish` 这个特殊的异常方法终止此次处理过程，而不再调用 ``write_error``。

对于404错误来说，可以在 `应用设置 <.Application.settings>` 中设置 默认的处理句柄类 - ``default_handler_class`` . 这个句柄类必须重写 `~.RequestHandler.prepare` 方法，而不是像 ``get()`` 等更详细的方法，这样它会在所有的HTTP 方法中都生效。 所以，像上面所说的，产生错误页面有两种方式：或者通过抛出一个 ``HTTPError(404)`` 异常并且重写 ``write_error`` 方法，或者调用 ``self.set_status(404)`` 方法并且在 ``prepare()`` 方法中直接产生返回数据（使用``self.write``）。

重定向
~~~~~~~~~~~

There are two main ways you can redirect requests in Tornado:
`.RequestHandler.redirect` and with the `.RedirectHandler`.

在Tornado中有两种主要的方式来进行重定向请求： `.RequestHandler.redirect` 方法和 使用 `.RedirectHandler`.

可以在一个 `.RequestHandler` 方法中使用 ``self.redirect()`` 将用户重定向到别处。此外，还有一个可选的参数
 `` permanent`` ，你可以用它来表明重定向是永久性的。 ``permanent`` 参数的默认值是 ``False`` ，生成一个 ``302 Found`` 的HTTP响应码，它比较适用于在用户 ``POST`` 请求成功后进行重定向之类的需求。如果 ``permanent`` 设置成 ``True`` ,将会生成 ``301 Moved Permanently`` 的HTTP响应码，某些情况下也是很有用的，比如使用搜索引擎友好的方式下将一个页面重定向到权威地址。

`.RedirectHandler` 类允许直接在 `.Application` 路由表中进行重定向配置。下面的例子展示了如何配置一个单向的静态重定向：

    app = tornado.web.Application([
        url(r"/app", tornado.web.RedirectHandler,
            dict(url="http://itunes.apple.com/my-app-id")),
        ])

`.RedirectHandler` 也支持正则表达式替换。下面的规则将所有 ``/pictures/`` 开头的请求重定向到 ``/photos/`` ::

    app = tornado.web.Application([
        url(r"/photos/(.*)", MyPhotoHandler),
        url(r"/pictures/(.*)", tornado.web.RedirectHandler,
            dict(url=r"/photos/\1")),
        ])

与 `.RequestHandler.redirect` 不同的是， `.RedirectHandler` 默认使用永久重定向。这是因为路由表不会在程序运行时改变，所以认为是永久性的，然而重定向也有可能是请求中其他逻辑改变的结果。 使用 `.RedirectHandler` 时将 ``permanent=False`` 加到 `.RedirectHandler` 的初始化参数中就可以产生一个临时的重定向了。

异步请求
~~~~~~~~~~~~~~~~~~~~~

Tornado的请求默认是同步的：当 ``get()``/``post()`` 方法返回的时候，请求被认为已经结束并且答复已经发出。由于在一个请求运行的时候，所有其他的请求都会被阻塞，因此长时间运行的请求应该异步化，以便于使用非阻塞的方式调用较慢的请求。在 :doc:`async` 中进行了更详细的阐述；这部分是对 `.RequestHandler` 子类中异步技术细节的相关说明。

使一个请求异步化的最简单的方式是使用 `.coroutine` 修饰器，它允许你使用  ``yield`` 关键字生成非阻塞I/O，并且在协程返回之前，不会有响应被发送。 查看 :doc:`coroutines` 文档查看更详细的资料。

在某些场景下，相比于回调为主导的方式，协程可能更不方便，在这种情况下，可以使用 `.tornado.web.asynchronous` 修饰器来替代。当这个修饰器被使用的时候，响应不会自动的发送；相反请求会保持开放状态，直到某些回调函数调用了 `.RequestHandler.finish` 方法。要确认在应用程序中调用了该方法，否则用户的浏览器会被hang住。

下面是一个使用Tornado的内置的异步HTTP客户端 `.AsyncHTTPClient` 调用 FriendFeed API的例子:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        # 使用@tornado.web.asynchronous异步修饰器
        @tornado.web.asynchronous
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            # 使用回调方式，http请求结束后，将结果作为参赛回调callback方法，既self.on_response
            http.fetch("http://friendfeed-api.com/v2/feed/bret",
                       callback=self.on_response)

        def on_response(self, response):
            if response.error: raise tornado.web.HTTPError(500)
            # Tornado中内置json解码函数，不需要使用json包
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")
            self.finish()

.. testoutput::
   :hide:

当 ``get()`` 方法返回的时候，本次请求还没有结束。当HTTP客户端最终调用 ``on_response()`` 方法的时候，本次请求仍然处于开放状态，最后调用 ``self.finish()`` 方法将响应结果刷到客户端。

为了便于比较，这里是一个使用协程相同的例子(貌似这个看起来还是更简单）:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        # 这里使用协程修饰器
        @tornado.gen.coroutine
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            # 使用yield而不是用回调
            response = yield http.fetch("http://friendfeed-api.com/v2/feed/bret")
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")

.. testoutput::
   :hide:

对于更高级的异步例子，可以查看 `聊天应用例子
<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>`_ ，它使用 `long polling
<http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_ 实现了一个AJAX的聊天室。
长轮询的用户可能需要去重写 ``on_connection_close()`` 方法在客户端关闭连接之后后来清理现场。（请注意查看该方法文档中的警告事项）
