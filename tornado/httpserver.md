- [简介](#%E7%AE%80%E4%BB%8B)
- [HTTPServer](#httpserver)
  - [HTTPServer.\_\_init\_\_](#httpserverinit)
  - [HTTPServer.listen](#httpserverlisten)
  - [HTTPServer.bind](#httpserverbind)
  - [HTTPServer.start](#httpserverstart)
  - [HTTPServer.stop](#httpserverstop)
  - [HTTPServer.\_handle_events](#httpserverhandleevents)
- [HTTPConnection](#httpconnection)
- [HTTPRequest](#httprequest)
- [版权](#%E7%89%88%E6%9D%83)

## 简介

HTTPServer 可以作为一个独立的 http 服务器使用,给它提供一个可调用对象，收到 request 时即调用该对象。  
对象中通过 write() 写入返回数据，通过 finish()结束返回：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from tornado import httpserver
from tornado import ioloop

def handle_request(request):
    message = "You requested %s\n" % request.uri
    request.write("HTTP/1.1 200 OK\r\nContent-Length: %d\r\n\r\n%s" % (
        len(message), message))
    request.finish()

http_server = httpserver.HTTPServer(handle_request)
http_server.listen(8888)
ioloop.IOLoop.instance().start()
```

```bash
$ http --verbose 'http://127.0.0.1:8888/hello'
GET /hello HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: 127.0.0.1:8888
User-Agent: HTTPie/0.9.2



HTTP/1.1 200 OK
Content-Length: 21

You requested /hello

```

## HTTPServer

非阻塞单线程 http 服务

### HTTPServer.\_\_init\_\_

保存 callback 回调对象

```python
class HTTPServer(object):
    def __init__(self, request_callback, no_keep_alive=False, io_loop=None,
                 xheaders=False, ssl_options=None):
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.io_loop = io_loop
        self.xheaders = xheaders
        self.ssl_options = ssl_options
        self._socket = None
        self._started = False
```

### HTTPServer.listen

1. 绑定端口
2. 监听
3. 注册到 ioloop

```python
    def listen(self, port, address=""):

        self.bind(port, address)
        self.start(1)
```

### HTTPServer.bind

绑定

```python
    def bind(self, port, address=""):
        """Binds this server to the given port on the given IP address.

        To start the server, call start(). If you want to run this server
        in a single process, you can call listen() as a shortcut to the
        sequence of bind() and start() calls.
        """
        assert not self._socket
        self._socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
        flags = fcntl.fcntl(self._socket.fileno(), fcntl.F_GETFD)
        flags |= fcntl.FD_CLOEXEC
        fcntl.fcntl(self._socket.fileno(), fcntl.F_SETFD, flags)
        self._socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self._socket.setblocking(0)
        self._socket.bind((address, port))
        self._socket.listen(128)
```

### HTTPServer.start

在 IOLoop 中启动监听服务  
将\_handle_events 函数注册到监听 socket 的 read 事件

```python
    def start(self, num_processes=1):
        assert not self._started
        self._started = True
        if num_processes is None or num_processes <= 0:
            num_processes = _cpu_count()
        if num_processes > 1 and ioloop.IOLoop.initialized():
            logging.error("Cannot run in multiple processes: IOLoop instance "
                          "has already been initialized. You cannot call "
                          "IOLoop.instance() before calling start()")
            num_processes = 1
        if num_processes > 1:
            logging.info("Pre-forking %d server processes", num_processes)
            for i in range(num_processes):
                if os.fork() == 0:
                    import random
                    from binascii import hexlify
                    try:
                        # If available, use the same method as
                        # random.py
                        seed = long(hexlify(os.urandom(16)), 16)
                    except NotImplementedError:
                        # Include the pid to avoid initializing two
                        # processes to the same value
                        seed(int(time.time() * 1000) ^ os.getpid())
                    random.seed(seed)
                    self.io_loop = ioloop.IOLoop.instance()
                    self.io_loop.add_handler(
                        self._socket.fileno(), self._handle_events,
                        ioloop.IOLoop.READ)
                    return
            # 0: os.WNOHANG 无子进程退出则不阻塞
            # 回收任意一个子 zombie进程
            os.waitpid(-1, 0)
        else:
            # 创建一个 io_loop 单例
            if not self.io_loop:
                self.io_loop = ioloop.IOLoop.instance()
            # 增加事件处理 handler
            self.io_loop.add_handler(self._socket.fileno(),
                                     self._handle_events,
                                     ioloop.IOLoop.READ)
```

### HTTPServer.stop

```python
    def stop(self):
        self.io_loop.remove_handler(self._socket.fileno())
        self._socket.close()
```

### HTTPServer.\_handle_events

注册到 ioloop  
read 事件产生(新请求进来)时调用

1. accept
2. 创建 HTTPConnection，将 request_callback 注册到 ioloop

```python
    def _handle_events(self, fd, events):
        while True:
            try:
                # 建立请求
                connection, address = self._socket.accept()
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise
            if self.ssl_options is not None:
                assert ssl, "Python 2.6+ and OpenSSL required for SSL"
                try:
                    connection = ssl.wrap_socket(connection,
                                                 server_side=True,
                                                 do_handshake_on_connect=False,
                                                 **self.ssl_options)
                except ssl.SSLError, err:
                    if err.args[0] == ssl.SSL_ERROR_EOF:
                        return connection.close()
                    else:
                        raise
                except socket.error, err:
                    if err.args[0] == errno.ECONNABORTED:
                        return connection.close()
                    else:
                        raise
            try:
                if self.ssl_options is not None:
                    stream = iostream.SSLIOStream(connection, io_loop=self.io_loop)
                else:
                    # 创建 stream 对象
                    stream = iostream.IOStream(connection, io_loop=self.io_loop)
                # 创建 HTTPConnection 对象
                # stream: stream对象
                # request_callback: 绑定的回调，新的请求来到时调用，一般是 Application 对象

                HTTPConnection(stream, address, self.request_callback,
                               self.no_keep_alive, self.xheaders)
            except:
                logging.error("Error in connection callback", exc_info=True)

```

## HTTPConnection

> Handles a connection to an HTTP client, executing HTTP requests.  
> We parse HTTP headers and bodies, and execute the request callback until the HTTP conection is closed.

1. 封装了一个 http 连接对象
2. 在初始化函数中完成 http 头的接收和解析
3. 完成 header 数据接收后，调用 `_on_headers` 函数
   创建 HTTPRequest 对象，并且以该 request 对象为参数调用 self.request_callback 方法，该方法绑定到 HTTPServer 的回调

```python
class HTTPConnection(object):
    # 设置 相关变量
    # read_until 触发 socket 事件回调链
    def __init__(self, stream, address, request_callback, no_keep_alive=False,
                 xheaders=False):
        self.stream = stream
        self.address = address
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.xheaders = xheaders
        self._request = None
        self._request_finished = False
        # Save stack context here, outside of any request.  This keeps
        # contexts from one request from leaking into the next.
        self._header_callback = stack_context.wrap(self._on_headers)
        self.stream.read_until("\r\n\r\n", self._header_callback)

    def write(self, chunk):
        assert self._request, "Request closed"
        if not self.stream.closed():
            self.stream.write(chunk, self._on_write_complete)

    def finish(self):
        assert self._request, "Request closed"
        self._request_finished = True
        if not self.stream.writing():
            self._finish_request()

    def _on_write_complete(self):
        if self._request_finished:
            self._finish_request()

    # 检查是否 keep-alive
    # 不是则关闭连接
    # 是则触发下一个 http 头读取事件
    def _finish_request(self):
        if self.no_keep_alive:
            disconnect = True
        else:
            connection_header = self._request.headers.get("Connection")
            if self._request.supports_http_1_1():
                disconnect = connection_header == "close"
            elif ("Content-Length" in self._request.headers
                    or self._request.method in ("HEAD", "GET")):
                disconnect = connection_header != "Keep-Alive"
            else:
                disconnect = True
        self._request = None
        self._request_finished = False
        if disconnect:
            self.stream.close()
            return
        self.stream.read_until("\r\n\r\n", self._header_callback)

    # 解析 http 头并构造 HTTPRequest
    def _on_headers(self, data):
        try:
            # 解析请求行
            eol = data.find("\r\n")
            start_line = data[:eol]
            try:
                method, uri, version = start_line.split(" ")
            except ValueError:
                raise _BadRequestException("Malformed HTTP request line")
            if not version.startswith("HTTP/"):
                raise _BadRequestException("Malformed HTTP version in HTTP Request-Line")
            # 解析请求头
            headers = httputil.HTTPHeaders.parse(data[eol:])
            self._request = HTTPRequest(
                connection=self, method=method, uri=uri, version=version,
                headers=headers, remote_ip=self.address[0])

            # 有 body 则读取
            content_length = headers.get("Content-Length")
            if content_length:
                content_length = int(content_length)
                if content_length > self.stream.max_buffer_size:
                    raise _BadRequestException("Content-Length too long")
                if headers.get("Expect") == "100-continue":
                    self.stream.write("HTTP/1.1 100 (Continue)\r\n\r\n")
                self.stream.read_bytes(content_length, self._on_request_body)
                return

            self.request_callback(self._request)
        except _BadRequestException, e:
            logging.info("Malformed HTTP request from %s: %s",
                         self.address[0], e)
            self.stream.close()
            return

    # POST PUT 方法检测表单域
    # extend 方法不会改变 dictinary 原来对象的值
    # 支持两种类型表单
    # application/x-www-form-urlencoded
    # multipart/form-data
    # 数据存放于 self._request.arguments
    def _on_request_body(self, data):
        self._request.body = data
        content_type = self._request.headers.get("Content-Type", "")
        if self._request.method in ("POST", "PUT"):
            if content_type.startswith("application/x-www-form-urlencoded"):
                arguments = cgi.parse_qs(self._request.body)
                for name, values in arguments.iteritems():
                    values = [v for v in values if v]
                    if values:
                        self._request.arguments.setdefault(name, []).extend(
                            values)
            elif content_type.startswith("multipart/form-data"):
                fields = content_type.split(";")
                for field in fields:
                    k, sep, v = field.strip().partition("=")
                    if k == "boundary" and v:
                        self._parse_mime_body(v, data)
                        break
                else:
                    logging.warning("Invalid multipart/form-data")
        self.request_callback(self._request)

    # 解析 multipart/form-data 格式表单
    # 目前只支持 Content-Disposition:form-data，不支持Content-Disposition:file
    # 不支持嵌套
    def _parse_mime_body(self, boundary, data):
        # The standard allows for the boundary to be quoted in the header,
        # although it's rare (it happens at least for google app engine
        # xmpp).  I think we're also supposed to handle backslash-escapes
        # here but I'll save that until we see a client that uses them
        # in the wild.
        if boundary.startswith('"') and boundary.endswith('"'):
            boundary = boundary[1:-1]
        if data.endswith("\r\n"):
            footer_length = len(boundary) + 6
        else:
            footer_length = len(boundary) + 4
        parts = data[:-footer_length].split("--" + boundary + "\r\n")
        for part in parts:
            if not part: continue
            eoh = part.find("\r\n\r\n")
            if eoh == -1:
                logging.warning("multipart/form-data missing headers")
                continue
            headers = httputil.HTTPHeaders.parse(part[:eoh])
            name_header = headers.get("Content-Disposition", "")
            if not name_header.startswith("form-data;") or \
               not part.endswith("\r\n"):
                logging.warning("Invalid multipart/form-data")
                continue
            value = part[eoh + 4:-2]
            name_values = {}
            for name_part in name_header[10:].split(";"):
                name, name_value = name_part.strip().split("=", 1)
                name_values[name] = name_value.strip('"').decode("utf-8")
            if not name_values.get("name"):
                logging.warning("multipart/form-data value missing name")
                continue
            name = name_values["name"]
            if name_values.get("filename"):
                ctype = headers.get("Content-Type", "application/unknown")
                self._request.files.setdefault(name, []).append(dict(
                    filename=name_values["filename"], body=value,
                    content_type=ctype))
            else:
                self._request.arguments.setdefault(name, []).append(value)
```

## HTTPRequest

封装 http 请求  
连接信息 HTTPConnection 保存在 connection 中
write、finish 方法给上层调用

```python
class HTTPRequest(object):
    """A single HTTP request.

    GET/POST arguments are available in the arguments property, which
    maps arguments names to lists of values (to support multiple values
    for individual names). Names and values are both unicode always.

    File uploads are available in the files property, which maps file
    names to list of files. Each file is a dictionary of the form
    {"filename":..., "content_type":..., "body":...}. The content_type
    comes from the provided HTTP header and should not be trusted
    outright given that it can be easily forged.

    An HTTP request is attached to a single HTTP connection, which can
    be accessed through the "connection" attribute. Since connections
    are typically kept open in HTTP/1.1, multiple requests can be handled
    sequentially on a single connection.
    """
    def __init__(self, method, uri, version="HTTP/1.0", headers=None,
                 body=None, remote_ip=None, protocol=None, host=None,
                 files=None, connection=None):
        self.method = method
        self.uri = uri
        self.version = version
        self.headers = headers or httputil.HTTPHeaders()
        self.body = body or ""
        if connection and connection.xheaders:
            # Squid uses X-Forwarded-For, others use X-Real-Ip
            self.remote_ip = self.headers.get(
                "X-Real-Ip", self.headers.get("X-Forwarded-For", remote_ip))
            # AWS uses X-Forwarded-Proto
            self.protocol = self.headers.get(
                "X-Scheme", self.headers.get("X-Forwarded-Proto", protocol))
            if self.protocol not in ("http", "https"):
                self.protocol = "http"
        else:
            self.remote_ip = remote_ip
            if protocol:
                self.protocol = protocol
            elif connection and isinstance(connection.stream,
                                           iostream.SSLIOStream):
                self.protocol = "https"
            else:
                self.protocol = "http"
        self.host = host or self.headers.get("Host") or "127.0.0.1"
        self.files = files or {}
        self.connection = connection
        self._start_time = time.time()
        self._finish_time = None

        scheme, netloc, path, query, fragment = urlparse.urlsplit(uri)
        self.path = path
        self.query = query
        arguments = cgi.parse_qs(query)
        self.arguments = {}
        for name, values in arguments.iteritems():
            values = [v for v in values if v]
            if values: self.arguments[name] = values

    def supports_http_1_1(self):
        """Returns True if this request supports HTTP/1.1 semantics"""
        return self.version == "HTTP/1.1"

    def write(self, chunk):
        """Writes the given chunk to the response stream."""
        assert isinstance(chunk, str)
        self.connection.write(chunk)

    def finish(self):
        """Finishes this HTTP request on the open connection."""
        self.connection.finish()
        self._finish_time = time.time()

    def full_url(self):
        """Reconstructs the full URL for this request."""
        return self.protocol + "://" + self.host + self.uri

    def request_time(self):
        """Returns the amount of time it took for this request to execute."""
        if self._finish_time is None:
            return time.time() - self._start_time
        else:
            return self._finish_time - self._start_time

    def get_ssl_certificate(self):
        """Returns the client's SSL certificate, if any.

        To use client certificates, the HTTPServer must have been constructed
        with cert_reqs set in ssl_options, e.g.:
            server = HTTPServer(app,
                ssl_options=dict(
                    certfile="foo.crt",
                    keyfile="foo.key",
                    cert_reqs=ssl.CERT_REQUIRED,
                    ca_certs="cacert.crt"))

        The return value is a dictionary, see SSLSocket.getpeercert() in
        the standard library for more details.
        http://docs.python.org/library/ssl.html#sslsocket-objects
        """
        try:
            return self.connection.stream.socket.getpeercert()
        except ssl.SSLError:
            return None

    def __repr__(self):
        attrs = ("protocol", "host", "method", "uri", "version", "remote_ip",
                 "body")
        args = ", ".join(["%s=%r" % (n, getattr(self, n)) for n in attrs])
        return "%s(%s, headers=%s)" % (
            self.__class__.__name__, args, dict(self.headers))
```

## 版权

作者：bigfish  
许可协议：[许可协议 知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)
