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

You can serve static files from Tornado by specifying the
``static_path`` setting in your application::

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

This setting will automatically make all requests that start with
``/static/`` serve from that static directory, e.g.,
``http://localhost:8888/static/foo.png`` will serve the file
``foo.png`` from the specified static directory. We also automatically
serve ``/robots.txt`` and ``/favicon.ico`` from the static directory
(even though they don't start with the ``/static/`` prefix).

In the above settings, we have explicitly configured Tornado to serve
``apple-touch-icon.png`` from the root with the `.StaticFileHandler`,
though it is physically in the static file directory. (The capturing
group in that regular expression is necessary to tell
`.StaticFileHandler` the requested filename; recall that capturing
groups are passed to handlers as method arguments.) You could do the
same thing to serve e.g. ``sitemap.xml`` from the site root. Of
course, you can also avoid faking a root ``apple-touch-icon.png`` by
using the appropriate ``<link />`` tag in your HTML.

To improve performance, it is generally a good idea for browsers to
cache static resources aggressively so browsers won't send unnecessary
``If-Modified-Since`` or ``Etag`` requests that might block the
rendering of the page. Tornado supports this out of the box with *static
content versioning*.

To use this feature, use the `~.RequestHandler.static_url` method in
your templates rather than typing the URL of the static file directly
in your HTML::

    <html>
       <head>
          <title>FriendFeed - {{ _("Home") }}</title>
       </head>
       <body>
         <div><img src="{{ static_url("images/logo.png") }}"/></div>
       </body>
     </html>

The ``static_url()`` function will translate that relative path to a URI
that looks like ``/static/images/logo.png?v=aae54``. The ``v`` argument
is a hash of the content in ``logo.png``, and its presence makes the
Tornado server send cache headers to the user's browser that will make
the browser cache the content indefinitely.

Since the ``v`` argument is based on the content of the file, if you
update a file and restart your server, it will start sending a new ``v``
value, so the user's browser will automatically fetch the new file. If
the file's contents don't change, the browser will continue to use a
locally cached copy without ever checking for updates on the server,
significantly improving rendering performance.

In production, you probably want to serve static files from a more
optimized static file server like `nginx <http://nginx.net/>`_. You
can configure most any web server to recognize the version tags used
by ``static_url()`` and set caching headers accordingly.  Here is the
relevant portion of the nginx configuration we use at FriendFeed::

    location /static/ {
        root /var/friendfeed/static;
        if ($query_string) {
            expires max;
        }
     }

.. _debug-mode:

Debug mode and automatic reloading
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you pass ``debug=True`` to the ``Application`` constructor, the app
will be run in debug/development mode. In this mode, several features
intended for convenience while developing will be enabled (each of which
is also available as an individual flag; if both are specified the
individual flag takes precedence):

* ``autoreload=True``: The app will watch for changes to its source
  files and reload itself when anything changes. This reduces the need
  to manually restart the server during development. However, certain
  failures (such as syntax errors at import time) can still take the
  server down in a way that debug mode cannot currently recover from.
* ``compiled_template_cache=False``: Templates will not be cached.
* ``static_hash_cache=False``: Static file hashes (used by the
  ``static_url`` function) will not be cached
* ``serve_traceback=True``: When an exception in a `.RequestHandler`
  is not caught, an error page including a stack trace will be
  generated.

Autoreload mode is not compatible with the multi-process mode of `.HTTPServer`.
You must not give `HTTPServer.start <.TCPServer.start>` an argument other than 1 (or
call `tornado.process.fork_processes`) if you are using autoreload mode.

The automatic reloading feature of debug mode is available as a
standalone module in `tornado.autoreload`.  The two can be used in
combination to provide extra robustness against syntax errors: set
``autoreload=True`` within the app to detect changes while it is running,
and start it with ``python -m tornado.autoreload myserver.py`` to catch
any syntax errors or other errors at startup.

Reloading loses any Python interpreter command-line arguments (e.g. ``-u``)
because it re-executes Python using `sys.executable` and `sys.argv`.
Additionally, modifying these variables will cause reloading to behave
incorrectly.

On some platforms (including Windows and Mac OSX prior to 10.6), the
process cannot be updated "in-place", so when a code change is
detected the old server exits and a new one starts.  This has been
known to confuse some IDEs.


WSGI and Google App Engine
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
