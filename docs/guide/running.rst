运行与部署
=====================

由于Tornado内置HTTP服务器，因此运行和部署与其他的Python web框架有些不同。与配置一个WSGI容器来运行应用不同的是，你需要写一个 ``main()`` 函数来开启应用。
# TODO: 需要了解一下WSGI的原理
Instead of configuring a WSGI container to find your application, you write a
``main()`` function that starts the server:

.. testcode::

    def main():
        # 创建应用
        app = make_app()
        # 监听端口
        app.listen(8888)
        # 开启IOLoop循环
        IOLoop.current().start()

    if __name__ == '__main__':
        main()

.. testoutput::
   :hide:

接下来需要配置操作系统或者程序管理器来开启服务端程序。请注意可能需要调大每一个进程的允许打开的最大文件句柄数量（以防止出现“Too many open files”的错误）。可以用以下几个方式来调大limit（比如调大到50000）：ulimit 命令；修改 /etc/security/limits.conf 文件或者通过修改进程监控程序中的 ``minfds`` 配置(比如supervisord）。

Processes and ports
进程和端口
~~~~~~~~~~~~~~~~~~~

由于Python GIL(Global Interpreter Lock)的原因，需要运行多个Python进程来充分利用多核CPU的性能。通常来说，最好是每个CPU一个进程。

Tornado包含一个内置的多进程模式可用来一次性开启多个进程。需要对标准的主函数做一点改造：

.. testcode::

    def main():
        # 创建应用
        app = make_app()
        # 创建server
        server = tornado.httpserver.HTTPServer(app)
        # 绑定端口
        server.bind(8888)
        # 每一个进程fork一个子进程
        server.start(0)  # forks one process per cpu
        # 开始IOLoop循环
        IOLoop.current().start()

.. testoutput::
   :hide:

#TODO: First, each
child process will have its own IOLoop, so it is important that
nothing touch the global IOLoop instance (even indirectly) before the
fork.  这段话需要理解一下。

这是开启多进程模式的最简单方式，并且所有的进程共享相同的端口，尽管它有一些限制。首先，每一个子进程将会拥有自己的IOLoop，所以务必注意在fork之前，不能与全局IOLoop实例有任何的直接接触（甚至是间接的也不可以）。其次，这种方式下很难做到零宕机更新。最后，由于所有的进程共享相同的端口，因此很难单独的进行监控。

对于更复杂的部署，建议单独的启动每一个进程，并且每个进程监听不同的端口。`supervisord <http://www.supervisord.org>`_ 的 "进程组（process groups）" 功能是一个很好的解决手段。当每个进程使用不同的端口时，就需要使用一个额外的负载均衡程序（如HAProxy或者ngingx）来向外部的访问者提供一个单一的地址（当然也可以在客户端内部进行负载均衡，比如简单的轮询或者求余）。

使用负载均衡程序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当使用一个负载均衡器的时候（如nginx），建议将 ``xheaders=True`` 传给 `.HTTPServer` 的构造函数。这样将会告知Tornado使用如 ``X-Real-IP`` 的 HTTP headers来获取真实的用户IP地址，而不是认为所有的流量来源都是负载均衡器的IP地址。

这是一个原始的nginx配置文件的结构是类似的
一个在FriendFeed使用我们

下面的这个原始的nginx的配置文件，结构与我们在FriendFeed使用的一个配置很相似。假定nginx和Tornado servers运行在相同的服务器上，并且运行4个Tornado servers，分别监听8000-8003四个端口::

    user nginx;
    worker_processes 1;

    error_log /var/log/nginx/error.log;
    pid /var/run/nginx.pid;

    events {
        worker_connections 1024;
        use epoll;
    }

    http {
        # 在upstream中列出所有的tornado server,当然如果你要做不同的路由跳转的时候可以定义多个upstream
        upstream frontends {
            server 127.0.0.1:8000;
            server 127.0.0.1:8001;
            server 127.0.0.1:8002;
            server 127.0.0.1:8003;
        }

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        access_log /var/log/nginx/access.log;

        keepalive_timeout 65;
        proxy_read_timeout 200;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1000;
        gzip_proxied any;
        gzip_types text/plain text/html text/css text/xml
                   application/x-javascript application/xml
                   application/atom+xml text/javascript;

        # Only retry if there was a communication error, not a timeout
        # on the Tornado server (to avoid propagating "queries of death"
        # to all frontends)
        proxy_next_upstream error;

        server {
            listen 80;

            # Allow file uploads
            client_max_body_size 50M;

            location ^~ /static/ {
                root /var/www;
                if ($query_string) {
                    expires max;
                }
            }
            location = /favicon.ico {
                rewrite (.*) /static/favicon.ico;
            }
            location = /robots.txt {
                rewrite (.*) /static/robots.txt;
            }

            location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_redirect off;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://frontends;
            }
        }
    }

静态文件和频繁访问文件的缓存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在Tornado中你可以通过在应用配置中指定 ``static_path`` 来提供静态文件服务::

    settings = {
        "static_path": os.path.join(os.path.dirname(__file__), "static"),
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
        (r"/(apple-touch-icon\.png)", tornado.web.StaticFileHandler,
         dict(path=settings['static_path'])),
    ], **settings)

这种配置将会自动的将所有以 ``/static/`` 开头的请求从设置的静态目录进行提供，例如： ``http://localhost:8888/static/foo.png`` 将会从指定的静态目录提供 ``foo.png`` 文件。我们也会自动的使用该静态目录提供 ``/robots.txt`` 和 ``/favicon.ico`` 文件（及时他们并不是 ``/static/`` 为前缀的请求）

上面的配置中，我们已经明确的配置Tornado使用 `.StaticFileHandler` 句柄来提供根目录下的 ``apple-touch-icon.png`` 文件，尽管它的物理位置是处于静态文件目录里。（在正则表达式中的捕获组必须要被请求的文件名告知 `.StaticFileHandler` 句柄；记住捕获组是作为方法参数传递给处理句柄的。）你可以按照类似的方式来提供一个根目录下 ``sitemap.xml`` 的文件请求。当然，你也可以通过在HTML中使用合适的 ``<link />`` 标签来避免伪造根目录的 ``apple-touch-icon.png`` 文件。（就是说可以通过/static/apple-touch-icon.png 路径来访问，而不是用/apple-touch-icon.png 路径）

为了提高性能，让浏览器主动缓存静态资源通常是一个好主意，这样浏览器就不需要再发送不必要的 ``If-Modified-Since`` 或者 ``Etag`` 请求（可能阻塞住页面渲染）。Tornado通过使用 *静态内容版本* 来支持这个功能。

使用这个功能，需要在你的模板文件中使用 `~.RequestHandler.static_url` 方法，而不是直接输入静态文件的URL。

    <html>
       <head>
          <title>FriendFeed - {{ _("Home") }}</title>
       </head>
       <body>
         <div><img src="{{ static_url("images/logo.png") }}"/></div>
       </body>
     </html>

``static_url()`` 函数会将相对路径转换成一个URI，例如： ``/static/images/logo.png?v=aae54`` 。参数 ``v`` 的值是文件 ``logo.png`` 内容的hash值，并且它的存在使得Tornado服务器将缓存headers发送到用户的浏览器，浏览器将会永久性的缓存相关内容。

由于参数 ``v`` 的值是基于文件内容，如果你更新了文件并且重启了服务器，它将发送一个新的 ``v`` 值，这样用户的浏览器将会自动获取新的文件。如果文件的内容没有改变，浏览器将会继续使用本地缓存的副本，而不会检查服务器端的更新，从而显著的提高渲染性能。

在生产过程中，你可能想要使用一个更优化的静态文件服务器提供静态文件服务，比如 `nginx <http://nginx.net/>`_ 。你可以配置几乎所有的Web服务器使用 ``static_url()`` 去识别版本标签，从而设置缓存响应headers。下面是一个我们在FriendFeed中使用的相关nginx配置片段：


    location /static/ {
        root /var/friendfeed/static;
        if ($query_string) {
            expires max;
        }
     }

.. _debug-mode:

调试模式和自动重载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果将 ``debug=True`` 传给 ``Application`` 的构造函数，应用将会以 调试/开发 模式运行。在这种模式下，为了方便，一些功能
将会启用（其中的每一个都可作为一个单独的标志；如果两者都指定了，单独的标志优先）:

* ``autoreload=True``: 应用将会观察源代码文件的变化，并且在它被改变的时候进行重载。在开发中将会减少很多手动重启服务器的次数。然后，需要注意如果在调试模式中进行代码更新的时候出现确切的错误（例如在import的时候发现语法错误），服务器将会宕机并且无法自动恢复。
* ``compiled_template_cache=False``: 模板不会被缓存。
* ``static_hash_cache=False``: 静态文件哈希表(使用的 ``static_url`` 函数)不会被缓存
* ``serve_traceback=True``: 当一个异常在一个 `.RequestHandler` 句柄中没有被捕获的时候，一个包含堆栈跟踪过程的错误页面将会生成。

#TODO:要了解代码架构后再来翻译这段
autoreload模式在 `.HTTPServer` 使用多进程模式的情况下不兼容。
Autoreload mode is not compatible with the multi-process mode of `.HTTPServer`.
You must not give `HTTPServer.start <.TCPServer.start>` an argument other than 1 (or
call `tornado.process.fork_processes`) if you are using autoreload mode.

#TODO:需要了解一下autoreload的原理
调试模式下的自动重载功能在 `tornado.autoreload` 可作为一个独立的模块使用。这两种方式可以相互配合来实现额外的稳定性来处理出现语法错误情况下的重载。在应用运行的时候，设置 ``autoreload=True`` 来探测代码改变，并且使用 ``python -m tornado.autoreload myserver.py`` 命令开启服务程序来启动时捕获任何语法或者其他错误。

#TODO:最后一句的理解，以及Python解释器参数的了解
重载过程会丢失所有的Python解释器的命令参数（比如 ``-u``) ,因为它使用 `sys.executable` 和 `sys.argv` 来重新执行Python程序。此外，修改这些变量？将引起重载行为异常。
Reloading loses any Python interpreter command-line arguments (e.g. ``-u``)
because it re-executes Python using `sys.executable` and `sys.argv`.
Additionally, modifying these variables will cause reloading to behave
incorrectly.

在一些平台上（包括Windows 以及 Mac OSX 10.6之前的版本），程序不能被"原地"更新，所以当代码发生改变的时候，旧进程会停止退出，并启动一个新的进程。这种方式将会使某些IDE产生困惑。


WSGI 和 Google App Engine
~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado is normally intended to be run on its own, without a WSGI
container.  However, in some environments (such as Google App Engine),
only WSGI is allowed and applications cannot run their own servers.
In this case Tornado supports a limited mode of operation that does
not support asynchronous operation but allows a subset of Tornado's
functionality in a WSGI-only environment.  The features that are
not allowed in WSGI mode include coroutines, the ``@asynchronous``
decorator, `.AsyncHTTPClient`, the ``auth`` module, and WebSockets.

You can convert a Tornado `.Application` to a WSGI application
with `tornado.wsgi.WSGIAdapter`.  In this example, configure
your WSGI container to find the ``application`` object:

.. testcode::

    import tornado.web
    import tornado.wsgi

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    tornado_app = tornado.web.Application([
        (r"/", MainHandler),
    ])
    application = tornado.wsgi.WSGIAdapter(tornado_app)

.. testoutput::
   :hide:

See the `appengine example application
<https://github.com/tornadoweb/tornado/tree/stable/demos/appengine>`_ for a
full-featured AppEngine app built on Tornado.
