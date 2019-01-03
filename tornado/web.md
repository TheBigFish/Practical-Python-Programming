## URLSpec

> Specifies mappings between URLs and handlers.

URLs 和 handlers 的映射关系

```python
class URLSpec(object):
    """Specifies mappings between URLs and handlers."""
    def __init__(self, pattern, handler_class, kwargs={}, name=None):
        """Creates a URLSpec.

        Parameters:

        pattern: Regular expression to be matched.  Any groups in the regex
            will be passed in to the handler's get/post/etc methods as
            arguments.

        handler_class: RequestHandler subclass to be invoked.

        kwargs (optional): A dictionary of additional arguments to be passed
            to the handler's constructor.

        name (optional): A name for this handler.  Used by
            Application.reverse_url.
        """
        if not pattern.endswith('$'):
            pattern += '$'
        self.regex = re.compile(pattern)
        self.handler_class = handler_class
        self.kwargs = kwargs
        self.name = name
        self._path, self._group_count = self._find_groups()

    def _find_groups(self):
        """Returns a tuple (reverse string, group count) for a url.

        For example: Given the url pattern /([0-9]{4})/([a-z-]+)/, this method
        would return ('/%s/%s/', 2).
        """

        # 返回正则表达式中 组的数量，以及以 /%s/对应的占位符
        pattern = self.regex.pattern
        if pattern.startswith('^'):
            pattern = pattern[1:]
        if pattern.endswith('$'):
            pattern = pattern[:-1]

        if self.regex.groups != pattern.count('('):
            # The pattern is too complicated for our simplistic matching,
            # so we can't support reversing it.
            return (None, None)

        pieces = []
        # 以 '(' 分割字符串，然后逐个匹配项查找 ')'，以此作为分组依据
        for fragment in pattern.split('('):
            if ')' in fragment:
                paren_loc = fragment.index(')')
                if paren_loc >= 0:
                    pieces.append('%s' + fragment[paren_loc + 1:])
            else:
                pieces.append(fragment)

        return (''.join(pieces), self.regex.groups)

    def reverse(self, *args):
        assert self._path is not None, \
            "Cannot reverse url regex " + self.regex.pattern
        assert len(args) == self._group_count, "required number of arguments "\
            "not found"
        if not len(args):
            return self._path
        # 使用列表推导
        return self._path % tuple([str(a) for a in args])
```

## Application

```python
class Application(object):
    """A collection of request handlers that make up a web application.

    Instances of this class are callable and can be passed directly to
    HTTPServer to serve the application::

        application = web.Application([
            (r"/", MainPageHandler),
        ])
        http_server = httpserver.HTTPServer(application)
        http_server.listen(8080)
        ioloop.IOLoop.instance().start()

    The constructor for this class takes in a list of URLSpec objects
    or (regexp, request_class) tuples. When we receive requests, we
    iterate over the list in order and instantiate an instance of the
    first request class whose regexp matches the request path.

    Each tuple can contain an optional third element, which should be a
    dictionary if it is present. That dictionary is passed as keyword
    arguments to the contructor of the handler. This pattern is used
    for the StaticFileHandler below::

        application = web.Application([
            (r"/static/(.*)", web.StaticFileHandler, {"path": "/var/www"}),
        ])

    We support virtual hosts with the add_handlers method, which takes in
    a host regular expression as the first argument::

        application.add_handlers(r"www\.myhost\.com", [
            (r"/article/([0-9]+)", ArticleHandler),
        ])

    You can serve static files by sending the static_path setting as a
    keyword argument. We will serve those files from the /static/ URI
    (this is configurable with the static_url_prefix setting),
    and we will serve /favicon.ico and /robots.txt from the same directory.

    .. attribute:: settings

       Additonal keyword arguments passed to the constructor are saved in the
       `settings` dictionary, and are often referred to in documentation as
       "application settings".
    """
    def __init__(self, handlers=None, default_host="", transforms=None,
                 wsgi=False, **settings):
        # 文件返回数据编码插件
        # Content-Encoding:gzip 压缩传输
        # Transfer-Encoding:chunck 块传输
        if transforms is None:
            self.transforms = []
            if settings.get("gzip"):
                self.transforms.append(GZipContentEncoding)
            self.transforms.append(ChunkedTransferEncoding)
        else:
            self.transforms = transforms

        self.handlers = []
        self.named_handlers = {}
        self.default_host = default_host
        self.settings = settings
        # 模板中使用
        self.ui_modules = {'linkify': _linkify,
                           'xsrf_form_html': _xsrf_form_html,
                           'Template': TemplateModule,
                           }
        self.ui_methods = {}
        self._wsgi = wsgi
        # 加载自定义 ui_modules ui_methods
        self._load_ui_modules(settings.get("ui_modules", {}))
        self._load_ui_methods(settings.get("ui_methods", {}))
        if self.settings.get("static_path"):
            path = self.settings["static_path"]
            handlers = list(handlers or [])
            static_url_prefix = settings.get("static_url_prefix",
                                             "/static/")
            # 使用 StaticFileHandler 处理静态文件
            handlers = [
                (re.escape(static_url_prefix) + r"(.*)", StaticFileHandler,
                 dict(path=path)),
                (r"/(favicon\.ico)", StaticFileHandler, dict(path=path)),
                (r"/(robots\.txt)", StaticFileHandler, dict(path=path)),
            ] + handlers
        # 注册静态文件路由
        if handlers: self.add_handlers(".*$", handlers)

        # Automatically reload modified modules
        # debug模式则打开模块自动加载
        if self.settings.get("debug") and not wsgi:
            import autoreload
            autoreload.start()

    def listen(self, port, address="", **kwargs):
        """Starts an HTTP server for this application on the given port.

        This is a convenience alias for creating an HTTPServer object
        and calling its listen method.  Keyword arguments not
        supported by HTTPServer.listen are passed to the HTTPServer
        constructor.  For advanced uses (e.g. preforking), do not use
        this method; create an HTTPServer and call its bind/start
        methods directly.

        Note that after calling this method you still need to call
        IOLoop.instance().start() to start the server.
        """
        # import is here rather than top level because HTTPServer
        # is not importable on appengine

        # 构造一个 HTTPServer 并进行侦听
        # 将自身作为请求回调传进去
        # 只是一个简便方法
        from tornado.httpserver import HTTPServer
        server = HTTPServer(self, **kwargs)
        server.listen(port, address)

    def add_handlers(self, host_pattern, host_handlers):
        """Appends the given handlers to our handler list.

        Note that host patterns are processed sequentially in the
        order they were added, and only the first matching pattern is
        used.  This means that all handlers for a given host must be
        added in a single add_handlers call.
        """
        if not host_pattern.endswith("$"):
            host_pattern += "$"
        handlers = []
        # The handlers with the wildcard host_pattern are a special
        # case - they're added in the constructor but should have lower
        # precedence than the more-precise handlers added later.
        # If a wildcard handler group exists, it should always be last
        # in the list, so insert new groups just before it.

        # .*$ 路由在路由表的最后面，优先匹配主机路由
        # 注意此处插入的是空路由，下面的循环中更新 handles，通过引用而同时更新
        if self.handlers and self.handlers[-1][0].pattern == '.*$':
            self.handlers.insert(-1, (re.compile(host_pattern), handlers))
        else:
            self.handlers.append((re.compile(host_pattern), handlers))

        for spec in host_handlers:
            if type(spec) is type(()):
                assert len(spec) in (2, 3)
                pattern = spec[0]
                handler = spec[1]
                if len(spec) == 3:
                    kwargs = spec[2]
                else:
                    kwargs = {}
                spec = URLSpec(pattern, handler, kwargs)
            handlers.append(spec)
            if spec.name:
                if spec.name in self.named_handlers:
                    logging.warning(
                        "Multiple handlers named %s; replacing previous value",
                        spec.name)
                self.named_handlers[spec.name] = spec

    def add_transform(self, transform_class):
        """Adds the given OutputTransform to our transform list."""
        self.transforms.append(transform_class)

    def _get_host_handlers(self, request):
        host = request.host.lower().split(':')[0]
        for pattern, handlers in self.handlers:
            if pattern.match(host):
                return handlers
        # Look for default host if not behind load balancer (for debugging)
        if "X-Real-Ip" not in request.headers:
            for pattern, handlers in self.handlers:
                if pattern.match(self.default_host):
                    return handlers
        return None

    def _load_ui_methods(self, methods):
        if type(methods) is types.ModuleType:
            self._load_ui_methods(dict((n, getattr(methods, n))
                                       for n in dir(methods)))
        elif isinstance(methods, list):
            for m in methods: self._load_ui_methods(m)
        else:
            for name, fn in methods.iteritems():
                if not name.startswith("_") and hasattr(fn, "__call__") \
                   and name[0].lower() == name[0]:
                    self.ui_methods[name] = fn

    def _load_ui_modules(self, modules):
        if type(modules) is types.ModuleType:
            self._load_ui_modules(dict((n, getattr(modules, n))
                                       for n in dir(modules)))
        elif isinstance(modules, list):
            for m in modules: self._load_ui_modules(m)
        else:
            assert isinstance(modules, dict)
            for name, cls in modules.iteritems():
                try:
                    if issubclass(cls, UIModule):
                        self.ui_modules[name] = cls
                except TypeError:
                    pass

    def __call__(self, request):
        """Called by HTTPServer to execute the request."""
        transforms = [t(request) for t in self.transforms]
        handler = None
        args = []
        kwargs = {}
        handlers = self._get_host_handlers(request)
        if not handlers:
            handler = RedirectHandler(
                self, request, url="http://" + self.default_host + "/")
        else:
            for spec in handlers:
                match = spec.regex.match(request.path)
                if match:
                    # None-safe wrapper around url_unescape to handle
                    # unmatched optional groups correctly
                    def unquote(s):
                        if s is None: return s
                        return escape.url_unescape(s, encoding=None)
                    handler = spec.handler_class(self, request, **spec.kwargs)
                    # Pass matched groups to the handler.  Since
                    # match.groups() includes both named and unnamed groups,
                    # we want to use either groups or groupdict but not both.
                    # Note that args are passed as bytes so the handler can
                    # decide what encoding to use.
                    kwargs = dict((k, unquote(v))
                                  for (k, v) in match.groupdict().iteritems())
                    if kwargs:
                        args = []
                    else:
                        args = [unquote(s) for s in match.groups()]
                    break
            if not handler:
                handler = ErrorHandler(self, request, status_code=404)

        # In debug mode, re-compile templates and reload static files on every
        # request so you don't need to restart to see changes
        if self.settings.get("debug"):
            if getattr(RequestHandler, "_templates", None):
                for loader in RequestHandler._templates.values():
                    loader.reset()
            RequestHandler._static_hashes = {}

        handler._execute(transforms, *args, **kwargs)
        return handler

    def reverse_url(self, name, *args):
        """Returns a URL path for handler named `name`

        The handler must be added to the application as a named URLSpec
        """
        if name in self.named_handlers:
            return self.named_handlers[name].reverse(*args)
        raise KeyError("%s not found in named urls" % name)

    def log_request(self, handler):
        """Writes a completed HTTP request to the logs.

        By default writes to the python root logger.  To change
        this behavior either subclass Application and override this method,
        or pass a function in the application settings dictionary as
        'log_function'.
        """
        if "log_function" in self.settings:
            self.settings["log_function"](handler)
            return
        if handler.get_status() < 400:
            log_method = logging.info
        elif handler.get_status() < 500:
            log_method = logging.warning
        else:
            log_method = logging.error
        request_time = 1000.0 * handler.request.request_time()
        log_method("%d %s %.2fms", handler.get_status(),
                   handler._request_summary(), request_time)
```
