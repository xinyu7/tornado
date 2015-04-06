.. title:: Tornado Web Server

.. meta::
    :google-site-verification: g4bVhgwbVO1d9apCUsT-eKlApg31Cygbp8VGZY8Rf0g

|Tornado Web Server|
====================

.. |Tornado Web Server| image:: tornado.png
    :alt: Tornado Web Server

`Tornado <http://www.tornadoweb.org>`_ 是一个最初由 `FriendFeed
<http://friendfeed.com>`_ 团队开发的Python web框架及异步网络类库。依托于非阻塞的网络I/O，Tronado可以支持数以万记的活跃链接，十分适用于如 `long polling <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_,
`WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_, 以及其他需要每个用户都保持一个活跃长连接的应用。

快速链接地址：
-----------

* |Download current version|: :current_tarball:`z` (:doc:`release notes <releases>`)
* `源代码地址 (github) <https://github.com/tornadoweb/tornado>`_
* 邮件列表: `讨论组 <http://groups.google.com/group/python-tornado>`_ and `通知 <http://groups.google.com/group/python-tornado-announce>`_
* `Stack Overflow <http://stackoverflow.com/questions/tagged/tornado>`_
* `维基百科 <https://github.com/tornadoweb/tornado/wiki/Links>`_

.. |Download current version| replace:: Download version |version|

Hello, world
------------

下面是一个使用Tornado的简单 "Hello, world" web应用的示例代码::

    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    application = tornado.web.Application([
        (r"/", MainHandler),
    ])

    if __name__ == "__main__":
        application.listen(8888)
        tornado.ioloop.IOLoop.current().start()

上面的例子中并没有使用Tornado任何的异步特性; 关注异步特性的可以看这个 `简单聊天室
<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>`_ 的例子。

安装
------------

**自动安装**::

    pip install tornado

Tornado 存在于 `PyPI <http://pypi.python.org/pypi/tornado>`_ 列表中，因此可以使用 ``pip`` or ``easy_install`` 进行安装。需要注意的是，当你使用上述方式安装Tronado时，并不包含源代码中包含的演示应用，所以如果你需要了解演示应用程序，最后下载一个源代码的最新副本。

*手动安装*: 下载 :current_tarball:`z`:

.. parsed-literal::

    tar xvzf tornado-|version|.tar.gz
    cd tornado-|version|
    python setup.py build
    sudo python setup.py install

Tronado的源代码目前托管在 `GitHub
<https://github.com/tornadoweb/tornado>`_ 上.

**先决条件**: Tornado 运行在Python 2.6, 2.7, 3.2, 3.3, and 3.4 版本上。 在任何版本上，都需要依赖 `certifi <https://pypi.python.org/pypi/certifi>`_ 包，在 Python 2版本上还需要依赖 `backports.ssl_match_hostname
<https://pypi.python.org/pypi/backports.ssl_match_hostname>`_ 包. 当你使用 ``pip`` or ``easy_install`` 安装Tornado的时候，上述依赖包将会自动被安装。如需要Tornado的一些其他特性，可能需要安装下面的可选类库：

* 在Python 2.6版本运行 Tornado的测试程序的时候需要依赖 `unittest2 <https://pypi.python.org/pypi/unittest2>`_ 包 (在最新的Python版本并不需要)。
* `concurrent.futures <https://pypi.python.org/pypi/futures>`_ 被推荐最为Tornado的线程池类库，引入后可以使用 `~tornado.netutil.ThreadedResolver` 类。Python 2版本需要额外安装，Python 3版本已经将该类库作为内部标准库。
* `pycurl <http://pycurl.sourceforge.net>`_ 在使用 ``tornado.curl_httpclient`` 类时需要使用。Libcurl的版本需要 7.18.2 或者更高的版本，最好使用 7.21.1以及更高的版本。
* `Twisted <http://www.twistedmatrix.com>`_ 包在使用 `tornado.platform.twisted` 功能时可能需要。
* `pycares <https://pypi.python.org/pypi/pycares>`_ 是一个非阻塞的DNS解析包，可以用来在不适合使用线程的地方来代替。
* `Monotime <https://pypi.python.org/pypi/Monotime>`_ 包提供了一个单调的时钟功能，提高了在时间频繁更新环境下的可靠性。在Python 3.3版本上并不需要。

**平台**: Tornado需要运行在任何类Unix的平台上，因此为实现最好的性能以及可扩展性，只有 Linux (支持 ``epoll``) 和 BSD (支持 ``kqueue``) 是推荐的生产部署环境（虽然 Mac OS X 系统起源于 BSD 并且支持 kqueue，但是它的网络性能普遍较差，因此仅推荐作为开发模式使用）。Tornado也可以运行在Windows系统上， 尽管这些配置并没有被正式支持，因此也仅推荐作为开发模式使用。 

文档
-------------

Tronado的文档拥有 `PDF 和 Epub 两种模式可供下载 <https://readthedocs.org/projects/tornado/downloads/>`_.

.. toctree::
   :titlesonly:

   入门指导（guide）
   web框架（webframework）
   http
   网络（networking）
   协程（coroutine）
   集成 （integration）
   通用（utilities）
   FAQ
   releases

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

讨论和支持
----------------------

你可以在 `Tornado 开发者邮件列表
<http://groups.google.com/group/python-tornado>`_ 中进行塔伦，也可以在 `GitHub 问题跟踪
<https://github.com/tornadoweb/tornado/issues>`_ 中汇报bug。其他的资源可以在 `Tornado 维基
<https://github.com/tornadoweb/tornado/wiki/Links>`_ 列表中查看。最新的发行版本会在 `公告邮件列表
<http://groups.google.com/group/python-tornado-announce>`_ 中进行通知。 

Tornado 是 `Facebook 的一个开源的技术
<http://developers.facebook.com/opensource/>`_ ，并在 `Apache License 2.0 
<http://www.apache.org/licenses/LICENSE-2.0.html>`_ 版本下发行。 

本网站和所有的文档都在 `Creative
Commons 3.0 <http://creativecommons.org/licenses/by/3.0/>`_ 下许可。 
