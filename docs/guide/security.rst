认证和安全
===========================

.. testsetup::

   import tornado.web

Cookies和安全cookie
~~~~~~~~~~~~~~~~~~~~~~~~~~

你可以使用 ``set_cookie`` 方法在用户的浏览器中设置cookies:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_cookie("mycookie"):
                self.set_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

.. testoutput::
   :hide:

Cookie是不安全的，可能很容易被客户端修改。如果你需要去设置cookies，比如认证当前的登陆用户，你需要去签名你的cookies以防止伪造。Tornado支持使用 `~.RequestHandler.set_secure_cookie` 和 `~.RequestHandler.get_secure_cookie` 方法签名cookies。使用这些方法，在创建应用的时候，需要指定一个叫做 ``cookie_secret`` 的私钥，并将该参数传递给application的配置。

.. testcode::

    application = tornado.web.Application([
        (r"/", MainHandler),
    ], cookie_secret="__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__")

.. testoutput::
   :hide:

被签名的cookies包含cookie的编码值以及一个时间戳和一个 `HMAC <http://en.wikipedia.org/wiki/HMAC>`_ 签名。如果cookie太旧或者签名不匹配的话， ``get_secure_cookie`` 函数将会返回一个 ``None`` 值，就像cookie并没有设置一样。安全版本的代码示例如下所示:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_secure_cookie("mycookie"):
                self.set_secure_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

.. testoutput::
   :hide:

Tornado的安全cookies保证了完整性，但无法保证保密性。这是因为，cookie不能被修改但是内容确实可以被用户查看的。私钥 ``cookie_secret`` 是一个对称的key，并且必须秘密保存，任何获取了该key值的人都可以伪造出自己的签名cookies。

默认情况下，Tornado的安全cookies在30天后过期。不过可以在 ``set_secure_cookie`` 方法中设置 ``expires_days`` 参数来改变cookies的过期时间，同时设置 ``get_secure_cookie`` 方法的 ``max_age_days`` 参数也可以达到类似的效果。这两个参数可以配合使用，比如在大部分需求中使用 ``expires_days`` 参数设置过期为30天，但是在一些敏感的行为（例如更改账户信息）中使用  ``max_age_days`` 参数得到一个更短的过期时间。

.. _user-authentication:

用户认证
~~~~~~~~~~~~~~~~~~~

The currently authenticated user is available in every request handler
as `self.current_user <.RequestHandler.current_user>`, and in every
template as ``current_user``. By default, ``current_user`` is
``None``.

To implement user authentication in your application, you need to
override the ``get_current_user()`` method in your request handlers to
determine the current user based on, e.g., the value of a cookie. Here
is an example that lets users log into the application simply by
specifying a nickname, which is then saved in a cookie:

.. testcode::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            return self.get_secure_cookie("user")

    class MainHandler(BaseHandler):
        def get(self):
            if not self.current_user:
                self.redirect("/login")
                return
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    class LoginHandler(BaseHandler):
        def get(self):
            self.write('<html><body><form action="/login" method="post">'
                       'Name: <input type="text" name="name">'
                       '<input type="submit" value="Sign in">'
                       '</form></body></html>')

        def post(self):
            self.set_secure_cookie("user", self.get_argument("name"))
            self.redirect("/")

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], cookie_secret="__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__")

.. testoutput::
   :hide:

You can require that the user be logged in using the `Python
decorator <http://www.python.org/dev/peps/pep-0318/>`_
`tornado.web.authenticated`. If a request goes to a method with this
decorator, and the user is not logged in, they will be redirected to
``login_url`` (another application setting). The example above could be
rewritten:

.. testcode::

    class MainHandler(BaseHandler):
        @tornado.web.authenticated
        def get(self):
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    settings = {
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

.. testoutput::
   :hide:

If you decorate ``post()`` methods with the ``authenticated``
decorator, and the user is not logged in, the server will send a
``403`` response.  The ``@authenticated`` decorator is simply
shorthand for ``if not self.current_user: self.redirect()`` and may
not be appropriate for non-browser-based login schemes.

Check out the `Tornado Blog example application
<https://github.com/tornadoweb/tornado/tree/stable/demos/blog>`_ for a
complete example that uses authentication (and stores user data in a
MySQL database).

Third party authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~

The `tornado.auth` module implements the authentication and
authorization protocols for a number of the most popular sites on the
web, including Google/Gmail, Facebook, Twitter, and FriendFeed.
The module includes methods to log users in via these sites and, where
applicable, methods to authorize access to the service so you can, e.g.,
download a user's address book or publish a Twitter message on their
behalf.

Here is an example handler that uses Google for authentication, saving
the Google credentials in a cookie for later access:

.. testcode::

    class GoogleOAuth2LoginHandler(tornado.web.RequestHandler,
                                   tornado.auth.GoogleOAuth2Mixin):
        @tornado.gen.coroutine
        def get(self):
            if self.get_argument('code', False):
                user = yield self.get_authenticated_user(
                    redirect_uri='http://your.site.com/auth/google',
                    code=self.get_argument('code'))
                # Save the user with e.g. set_secure_cookie
            else:
                yield self.authorize_redirect(
                    redirect_uri='http://your.site.com/auth/google',
                    client_id=self.settings['google_oauth']['key'],
                    scope=['profile', 'email'],
                    response_type='code',
                    extra_params={'approval_prompt': 'auto'})

.. testoutput::
   :hide:

See the `tornado.auth` module documentation for more details.

.. _xsrf:

Cross-site request forgery protection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Cross-site request
forgery <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_, or
XSRF, is a common problem for personalized web applications. See the
`Wikipedia
article <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_ for
more information on how XSRF works.

The generally accepted solution to prevent XSRF is to cookie every user
with an unpredictable value and include that value as an additional
argument with every form submission on your site. If the cookie and the
value in the form submission do not match, then the request is likely
forged.

Tornado comes with built-in XSRF protection. To include it in your site,
include the application setting ``xsrf_cookies``:

.. testcode::

    settings = {
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

.. testoutput::
   :hide:

If ``xsrf_cookies`` is set, the Tornado web application will set the
``_xsrf`` cookie for all users and reject all ``POST``, ``PUT``, and
``DELETE`` requests that do not contain a correct ``_xsrf`` value. If
you turn this setting on, you need to instrument all forms that submit
via ``POST`` to contain this field. You can do this with the special
`.UIModule` ``xsrf_form_html()``, available in all templates::

    <form action="/new_message" method="post">
      {% module xsrf_form_html() %}
      <input type="text" name="message"/>
      <input type="submit" value="Post"/>
    </form>

If you submit AJAX ``POST`` requests, you will also need to instrument
your JavaScript to include the ``_xsrf`` value with each request. This
is the `jQuery <http://jquery.com/>`_ function we use at FriendFeed for
AJAX ``POST`` requests that automatically adds the ``_xsrf`` value to
all requests::

    function getCookie(name) {
        var r = document.cookie.match("\\b" + name + "=([^;]*)\\b");
        return r ? r[1] : undefined;
    }

    jQuery.postJSON = function(url, args, callback) {
        args._xsrf = getCookie("_xsrf");
        $.ajax({url: url, data: $.param(args), dataType: "text", type: "POST",
            success: function(response) {
            callback(eval("(" + response + ")"));
        }});
    };

For ``PUT`` and ``DELETE`` requests (as well as ``POST`` requests that
do not use form-encoded arguments), the XSRF token may also be passed
via an HTTP header named ``X-XSRFToken``.  The XSRF cookie is normally
set when ``xsrf_form_html`` is used, but in a pure-Javascript application
that does not use any regular forms you may need to access
``self.xsrf_token`` manually (just reading the property is enough to
set the cookie as a side effect).

If you need to customize XSRF behavior on a per-handler basis, you can
override `.RequestHandler.check_xsrf_cookie()`. For example, if you
have an API whose authentication does not use cookies, you may want to
disable XSRF protection by making ``check_xsrf_cookie()`` do nothing.
However, if you support both cookie and non-cookie-based authentication,
it is important that XSRF protection be used whenever the current
request is authenticated with a cookie.
