模板和UI
================

.. testsetup::

   import tornado.web

Tornado包含一个简单、快速、灵活的模板语言。本节介绍了模板语言以及相关问题，如国际化等。

Tornado也可以喝其他的Python模板语言配合使用，然而并没有将这些系统整合到 `.RequestHandler.render` 方法。而且简单的将模板渲染成字符串，然后传递给 `.RequestHandler.write` 方法。

配置模板
~~~~~~~~~~~~~~~~~~~~~

默认情况下，Tornado会在当前被引用的 ``.py`` 文件所在的相同目录寻找模板文件。可以在 `Application setting
<.Application.settings>` 中设置 ``template_path`` 参数来让Tornado到设定的目录中寻找模板文件（如果你为不同的处理句柄设置不同的模板文件目录的化，可以通过重写 `.RequestHandler.get_template_path` 方法来解决）。


#TODO: 这段需要再理解一下
To load templates from a non-filesystem location, subclass
`tornado.template.BaseLoader` and pass an instance as the
``template_loader`` application setting.

如果要在一个非文件系统的位置加载模板，需要写一个 `tornado.template.BaseLoader` 的子类，并且在应用配置中设置 ``template_loader`` 参数的值为该子类的实例。

默认情况下编译完成后的模板将会被缓存；可以通过在应用设置（application settings）中设置参数 ``compiled_template_cache=False`` 或 ``debug=True`` 来关闭缓存，这样每次模板被更改后都会自动加载。

模板语法
~~~~~~~~~~~~~~~

一个Tornado的模板其实就是将Python的控制序列和表达式嵌入标记内的HTML文件（或者任何其他基于文本的格式）::

    <html>
       <head>
          <title>{{ title }}</title>
       </head>
       <body>
         <ul>
           {% for item in items %}
             <li>{{ escape(item) }}</li>
           {% end %}
         </ul>
       </body>
     </html>

如果你将这个模板文件保存为： "template.html" ，并且将其放在你的Python文件的相同目录内，你就可以用下面的方式进行模板渲染了:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            items = ["Item 1", "Item 2", "Item 3"]
            # title,items分别是上面的template.html模板中的变量。
            self.render("template.html", title="My title", items=items)

.. testoutput::
   :hide:

Tornado的模板支持 *控制语句* 和 *表达式* 。控制语句使用 ``{%`` and ``%}`` 环绕，例如: ``{% if len(items) > 2 %}`` 。表达式使用 ``{{`` 和 ``}}`` 来环绕, 比如: ``{{ items[0] }}`` 。

控制语句大致与Python语句类似。它支持 ``if``, ``for``, ``while``, 和 ``try`` ，并且都必须使用 ``{% end %}`` 标志来结束。它也可以使用 ``extends`` 和 ``block`` 标签来实现 *模板继承* ，这些都在 `tornado.template` 文档中有详细的介绍。

表达式可也是任意的Python表达式，包括函数调用。模板中的代码会被执行在一个包括羡慕的对象和功能的命名空间中（需要注意的是：这些列出用来进行模板渲染的条目适用于在使用 `.RequestHandler.render` 和 `~.RequestHandler render_string` 的时候。如果你直接在 `.RequestHandler` 外使用 `tornado.template` 模块，这些条目很多都是不存在的）。

- ``escape``: `tornado.escape.xhtml_escape` 的别名
- ``xhtml_escape``: `tornado.escape.xhtml_escape` 的别名
- ``url_escape``: `tornado.escape.url_escape` 的别名
- ``json_encode``: `tornado.escape.json_encode` 的别名
- ``squeeze``:  `tornado.escape.squeeze` 的别名
- ``linkify``:  `tornado.escape.linkify`
- ``datetime``: Python 的 `datetime` 模块
- ``handler``: 当前的 `.RequestHandler` 对象
- ``request``:  `handler.request <.HTTPServerRequest>` 的别名
- ``current_user``: `handler.current_user <.RequestHandler.current_user>` 的别名
- ``locale``: `handler.locale <.Locale>` 的别名
- ``_``: `handler.locale.translate <.Locale.translate>` 的别名
- ``static_url``: `handler.static_url <.RequestHandler.static_url>` 的别名
- ``xsrf_form_html``: `handler.xsrf_form_html <.RequestHandler.xsrf_form_html>` 的别名
- ``reverse_url``: `.Application.reverse_url` 的别名
- 所有来自`` ui_methods``项和` ` ui_modules`` 的 `` Application``设置
- 所有传给 `~.RequestHandler.render` 或 `~.RequestHandler.render_string` 方法的关键字

当你使用Tornado创建一个真正的应用的时候，你会想去使用Tornado模板的全部功能，尤其是模板继承。 可以阅读 `tornado.template` 章节来了解详细的功能（包括 ``UIModules`` 的一些功能是在 `tornado.web` 模块中实现的）。

在引擎之中，Tornado的模板会被直接翻译成Python代码。在模板中的你所引入的表达式会一字不差的复制成一个Python函数来代替你的模板。在模板语言中，我们不会有任何限制；我们意在创造一个灵活的模板系统，而不是一个有各种限制的模板系统。因此，如果你在模板的表达式中随便写各种代码，当你执行模板的时候就会产生各种各样的错误了。

默认情况下，所有的模板输出的时候都会使用 `tornado.escape.xhtml_escape` 函数加密。这个行为可以通过传递 ``autoescape=None`` 给 `.Application` 设置或者 `.tornado.template.Loader` 的构造函数来关闭，也可以使用 ``{% autoescape None %}`` 命令仅在某个模板文件中关闭该功能，或者使用 ``{% raw ...%}`` 替换 ``{{ ... }}`` 从而在某个单独的表达式中关闭该功能。

#TODO:最后一句理解的不好
需要注意，虽然Tornado自动的模板加密有助于避免 XSS 漏洞，然而并不能解决所有的情况。出现在特定的位置的表达式，可能需要额外的转义，例如在Javascript或CSS里。此外，还有其他几点务必要注意：要一直使用双引号、在HTML属性中的 `.xhtml_escape` 可能包含其他未信任的内容，还有一个单独的escaping 函数必须被attributes 使用。（点击 http://wonko.com/post/html-escaping 查看详细内容）。


国际化
~~~~~~~~~~~~~~~~~~~~

当前用户的语言环境（无论用户是否登录）总是可用的，比如在请求handler中使用 ``self.locale`` 或在模板中使用 ``locale`` 。语言的名字（比如 ``en_US`` ）可以使用 ``locale.name`` 获取，并且你可以使用 `.Locale.translate` 方法来翻译字符串。 模板中也有全局的函数 ``_()`` 用来进行字符串翻译。翻译函数有两种形式::

    _("Translate this string")

这种是基于当前的语言环境直接翻译字符串内容，以及::

    _("A person liked this", "%(num)d people liked this", len(people)) % {"num": len(people)}

这种翻译字符串内容的方式是基于第三个参数来确定是单数还是复数形式。在上面的例子中，如果 ``len(people)`` 的结果是 ``1`` ，就会按照第一个字符串翻译；如果是 ``len(people)`` 的值是复数则会按照第二个字符串翻译。


The most common pattern for translations is to use Python named
placeholders for variables (the ``%(num)d`` in the example above) since
placeholders can move around on translation.

这是一个正确的国际化的模板::

    <html>
       <head>
          <title>FriendFeed - {{ _("Sign in") }}</title>
       </head>
       <body>
         <form action="{{ request.path }}" method="post">
           <div>{{ _("Username") }} <input type="text" name="username"/></div>
           <div>{{ _("Password") }} <input type="password" name="password"/></div>
           <div><input type="submit" value="{{ _("Sign in") }}"/></div>
           {% module xsrf_form_html() %}
         </form>
       </body>
     </html>

默认情况下我们会通过用户浏览器发送的 ``Accept-Language`` header信息获取用户的语言环境。如果无法获得合适的 ``Accept-Language`` 值，会默认使用 ``en_US`` 。如果想让用户自己设置偏好的语言环境，你可以通过重写 `.RequestHandler.get_user_locale` 方法来覆盖默认的语言环境:

.. testcode::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            user_id = self.get_secure_cookie("user")
            if not user_id: return None
            return self.backend.get_user_by_id(user_id)

        def get_user_locale(self):
            if "locale" not in self.current_user.prefs:
                # Use the Accept-Language header
                return None
            return self.current_user.prefs["locale"]

.. testoutput::
   :hide:

如果 ``get_user_locale`` 函数返回了 ``None`` 值，我们会重新查询 ``Accept-Language`` header的值。

#TODO:这段后续再修复一下
The `tornado.locale` module supports loading translations in two
formats: the ``.mo`` format used by `gettext` and related tools, and a
simple ``.csv`` format.  An application will generally call either
`tornado.locale.load_translations` or
`tornado.locale.load_gettext_translations` once at startup; see those
methods for more details on the supported formats..

#TODO:最后那句是看源码么？
`tornado.locale`  模板支持两种形式的文件来加载翻译： ``.mo`` 模式可以通过 `gettext` 和相关的工具来使用，以及一个简单的 ``.csv`` 格式。通常情况下应用会在启动的时候调用一次 `tornado.locale.load_translations` 或者 `tornado.locale.load_gettext_translations` 方法中的一个。
可以查看这些方法以便获取更多支持格式的细节...（see those methods for more details on the supported formats.. ）

你可以在应用中使用 `tornado.locale.get_supported_locales()` 函数获取所支持的语言环境列表。用户的语言环境会被选择成最接近支持的语言环境中的一个。比如，如果用户的语言环境是 ``es_GT`` ，并且 ``es`` 这个语言环境是支持的，那么 ``self.locale`` 将会使用 ``es`` 作为当前语言环境去处理请求。如果没有较接近的语言环境被匹配，则会使用 ``en_US`` 作为默认的语言环境。

.. _ui-modules:

UI 模块
~~~~~~~~~~

Tornado supports *UI modules* to make it easy to support standard,
reusable UI widgets across your application. UI modules are like special
function calls to render components of your page, and they can come
packaged with their own CSS and JavaScript.

For example, if you are implementing a blog, and you want to have blog
entries appear on both the blog home page and on each blog entry page,
you can make an ``Entry`` module to render them on both pages. First,
create a Python module for your UI modules, e.g., ``uimodules.py``::

    class Entry(tornado.web.UIModule):
        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", entry=entry, show_comments=show_comments)

Tell Tornado to use ``uimodules.py`` using the ``ui_modules`` setting in
your application::

    from . import uimodules

    class HomeHandler(tornado.web.RequestHandler):
        def get(self):
            entries = self.db.query("SELECT * FROM entries ORDER BY date DESC")
            self.render("home.html", entries=entries)

    class EntryHandler(tornado.web.RequestHandler):
        def get(self, entry_id):
            entry = self.db.get("SELECT * FROM entries WHERE id = %s", entry_id)
            if not entry: raise tornado.web.HTTPError(404)
            self.render("entry.html", entry=entry)

    settings = {
        "ui_modules": uimodules,
    }
    application = tornado.web.Application([
        (r"/", HomeHandler),
        (r"/entry/([0-9]+)", EntryHandler),
    ], **settings)

Within a template, you can call a module with the ``{% module %}``
statement.  For example, you could call the ``Entry`` module from both
``home.html``::

    {% for entry in entries %}
      {% module Entry(entry) %}
    {% end %}

and ``entry.html``::

    {% module Entry(entry, show_comments=True) %}

Modules can include custom CSS and JavaScript functions by overriding
the ``embedded_css``, ``embedded_javascript``, ``javascript_files``, or
``css_files`` methods::

    class Entry(tornado.web.UIModule):
        def embedded_css(self):
            return ".entry { margin-bottom: 1em; }"

        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", show_comments=show_comments)

Module CSS and JavaScript will be included once no matter how many times
a module is used on a page. CSS is always included in the ``<head>`` of
the page, and JavaScript is always included just before the ``</body>``
tag at the end of the page.

When additional Python code is not required, a template file itself may
be used as a module. For example, the preceding example could be
rewritten to put the following in ``module-entry.html``::

    {{ set_resources(embedded_css=".entry { margin-bottom: 1em; }") }}
    <!-- more template html... -->

This revised template module would be invoked with::

    {% module Template("module-entry.html", show_comments=True) %}

The ``set_resources`` function is only available in templates invoked
via ``{% module Template(...) %}``. Unlike the ``{% include ... %}``
directive, template modules have a distinct namespace from their
containing template - they can only see the global template namespace
and their own keyword arguments.
