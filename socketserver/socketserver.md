# SocketServer.py

> Creating network servers.

## contents

- [SocketServer.py](#socketserverpy)
  - [contents](#contents)
  - [file head](#file-head)
  - [BaseServer](#baseserver)
    - [BaseServer.serve_forever](#baseserverserveforever)
    - [BaseServer.shutdown](#baseservershutdown)
    - [BaseServer.handle_request](#baseserverhandlerequest)
    - [BaseServer.\_handle_request_noblock](#baseserverhandlerequestnoblock)
    - [BaseServer Overridden functions](#baseserver-overridden-functions)
  - [TCPServer](#tcpserver)
  - [UDPServer](#udpserver)
  - [ForkingMixIn](#forkingmixin)
  - [ThreadingMixIn](#threadingmixin)
  - [BaseRequestHandler](#baserequesthandler)
  - [StreamRequestHandler](#streamrequesthandler)
  - [DatagramRequestHandler](#datagramrequesthandler)
  - [版权](#%E7%89%88%E6%9D%83)

## file head

```python

__version__ = "0.4"


import socket
import select
import sys
import os
import errno
try:
    import threading
except ImportError:
    import dummy_threading as threading

__all__ = ["TCPServer","UDPServer","ForkingUDPServer","ForkingTCPServer",
           "ThreadingUDPServer","ThreadingTCPServer","BaseRequestHandler",
           "StreamRequestHandler","DatagramRequestHandler",
           "ThreadingMixIn", "ForkingMixIn"]
if hasattr(socket, "AF_UNIX"):
    __all__.extend(["UnixStreamServer","UnixDatagramServer",
                    "ThreadingUnixStreamServer",
                    "ThreadingUnixDatagramServer"])

# 出现 EINTR 则重新调用
def _eintr_retry(func, *args):
    """restart a system call interrupted by EINTR"""
    while True:
        try:
            return func(*args)
        except (OSError, select.error) as e:
            if e.args[0] != errno.EINTR:
                raise
```

## BaseServer

RequestHandlerClass 注册 handle 函数。  
finish_request 中实例化，调用用户定义的 handle 函数

```python
class BaseServer:
    timeout = None

    def __init__(self, server_address, RequestHandlerClass):
        """Constructor.  May be extended, do not override."""
        self.server_address = server_address
        self.RequestHandlerClass = RequestHandlerClass
        self.__is_shut_down = threading.Event()
        self.__shutdown_request = False

    def server_activate(self):
        """Called by constructor to activate the server.

        May be overridden.

        """
        pass
```

### BaseServer.serve_forever

服务循环

1. 监听端口
2. 处理请求

```python
    def serve_forever(self, poll_interval=0.5):
        """Handle one request at a time until shutdown.

        Polls for shutdown every poll_interval seconds. Ignores
        self.timeout. If you need to do periodic tasks, do them in
        another thread.
        """
        self.__is_shut_down.clear()
        try:
            while not self.__shutdown_request:
                # 调用 select 监视请求，处理 EINTR 异常
                r, w, e = _eintr_retry(select.select, [self], [], [],
                                       poll_interval)
                # 有请求进来
                if self in r:
                    self._handle_request_noblock()
        finally:
            self.__shutdown_request = False
            self.__is_shut_down.set()
```

### BaseServer.shutdown

停止 serve_forever 循环.  
`__is_shut_down` 通知外部，循环已经退出  
注意 threading.Event() 的用法，只设置一次，避免使用 Event 进行频繁的设置/清除。
需要在与 serve_forever 不同的线程中调用.  
因为调用 shutdown 后需要 wait 信号量，程序会 block，block 后 serve_forever 无法执行  
serve_forever 收到请求后才能退出设置信号量

**注意**  
`self.__shutdown_request` 的读写操作，属于原子操作，在多线程中使用是安全的

```python
    def shutdown(self):
        """Stops the serve_forever loop.

        Blocks until the loop has finished. This must be called while
        serve_forever() is running in another thread, or it will
        deadlock.
        """
        self.__shutdown_request = True
        self.__is_shut_down.wait()
```

### BaseServer.handle_request

和 serve_forever 并列的函数
如果不调用 server_forever, 在外面循环调用 handle_request

```python
    # The distinction between handling, getting, processing and
    # finishing a request is fairly arbitrary.  Remember:
    #
    # - handle_request() is the top-level call.  It calls
    #   select, get_request(), verify_request() and process_request()
    # - get_request() is different for stream or datagram sockets
    # - process_request() is the place that may fork a new process
    #   or create a new thread to finish the request
    # - finish_request() instantiates the request handler class;
    #   this constructor will handle the request all by itself

    def handle_request(self):
        """Handle one request, possibly blocking.

        Respects self.timeout.
        """
        # Support people who used socket.settimeout() to escape
        # handle_request before self.timeout was available.

        # 如果用户使用 socket.settimeout() 设置了超时时间，则选取一个小的
        timeout = self.socket.gettimeout()
        if timeout is None:
            timeout = self.timeout
        elif self.timeout is not None:
            timeout = min(timeout, self.timeout)
        # select,监听连接，会阻塞直到超时
        fd_sets = _eintr_retry(select.select, [self], [], [], timeout)
        if not fd_sets[0]:
            self.handle_timeout()
            return
        # 处理请求
        self._handle_request_noblock()
```

### BaseServer.\_handle_request_noblock

真正的请求处理函数

1. get_request: 接收请求 accept
2. verify_request: 验证，做一些验证工作，比如 ip 过滤
3. process_request: 处理请求，子类重写该方法后，需要 调用 SocketServer.BaseServer.process_request,
4. BaseServer.process_request 中有 BaseRequestHandler 的回调动作,实例化用户定义的 handler, `__init__` 中完成对 `handle()` 的调用
5. shutdown_reques: 关闭连接

```python

    def _handle_request_noblock(self):
        """Handle one request, without blocking.

        I assume that select.select has returned that the socket is
        readable before this function was called, so there should be
        no risk of blocking in get_request().
        """
        try:
            # 接收请求
            # get_request 由子类实现，一般为接收请求，返回 socket
            request, client_address = self.get_request()
        except socket.error:
            return
        if self.verify_request(request, client_address):
            try:
                self.process_request(request, client_address)
            except:
                self.handle_error(request, client_address)
                self.shutdown_request(request)
        else:
            self.shutdown_request(request)
```

### BaseServer Overridden functions

```python
    def handle_timeout(self):
        """Called if no new request arrives within self.timeout.

        Overridden by ForkingMixIn.
        """
        pass

    def verify_request(self, request, client_address):
        """Verify the request.  May be overridden.

        Return True if we should proceed with this request.

        """
        return True

    def process_request(self, request, client_address):
        """Call finish_request.

        Overridden by ForkingMixIn and ThreadingMixIn.

        """
        self.finish_request(request, client_address)
        self.shutdown_request(request)

    def server_close(self):
        """Called to clean-up the server.

        May be overridden.

        """
        pass

    def finish_request(self, request, client_address):
        """Finish one request by instantiating RequestHandlerClass."""
        self.RequestHandlerClass(request, client_address, self)

    def shutdown_request(self, request):
        """Called to shutdown and close an individual request."""
        self.close_request(request)

    def close_request(self, request):
        """Called to clean up an individual request."""
        pass

    def handle_error(self, request, client_address):
        """Handle an error gracefully.  May be overridden.

        The default is to print a traceback and continue.

        """
        print '-'*40
        print 'Exception happened during processing of request from',
        print client_address
        import traceback
        traceback.print_exc() # XXX But this goes to stderr!
        print '-'*40
```

## TCPServer

shutdown_request 先调用 socket.shutdown 后调用 socket.close

> - close()releases the resource associated with a connection but does not necessarily close the connection immediately. If you want to close the connection in a timely fashion, callshutdown() beforeclose().
> - Shut down one or both halves of the connection. If how is SHUT_RD, further receives are disallowed. If how is SHUT_WR, further sends are disallowed. Ifhow is SHUT_RDWR, further sends and receives are disallowed. Depending on the platform, shutting down one half of the connection can also close the opposite half (e.g. on Mac OS X, shutdown(SHUT_WR) does not allow further reads on the other end of the connection).

```python
class TCPServer(BaseServer):

    address_family = socket.AF_INET

    socket_type = socket.SOCK_STREAM

    request_queue_size = 5

    allow_reuse_address = False

    def __init__(self, server_address, RequestHandlerClass, bind_and_activate=True):
        """Constructor.  May be extended, do not override."""
        BaseServer.__init__(self, server_address, RequestHandlerClass)
        self.socket = socket.socket(self.address_family,
                                    self.socket_type)
        if bind_and_activate:
            try:
                self.server_bind()
                self.server_activate()
            except:
                self.server_close()
                raise

    def server_bind(self):
        """Called by constructor to bind the socket.

        May be overridden.

        """
        if self.allow_reuse_address:
            self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.socket.bind(self.server_address)
        self.server_address = self.socket.getsockname()

    def server_activate(self):
        """Called by constructor to activate the server.

        May be overridden.

        """
        self.socket.listen(self.request_queue_size)

    def server_close(self):
        """Called to clean-up the server.

        May be overridden.

        """
        self.socket.close()

    def fileno(self):
        """Return socket file number.

        Interface required by select().

        """
        return self.socket.fileno()

    def get_request(self):
        """Get the request and client address from the socket.

        May be overridden.

        """
        return self.socket.accept()

    # 调用 shutdown 后调用 close，立即关闭并释放资源
    def shutdown_request(self, request):
        """Called to shutdown and close an individual request."""
        try:
            #explicitly shutdown.  socket.close() merely releases
            #the socket and waits for GC to perform the actual close.
            request.shutdown(socket.SHUT_WR)
        except socket.error:
            pass #some platforms may raise ENOTCONN here
        self.close_request(request)

    def close_request(self, request):
        """Called to clean up an individual request."""
        request.close()
```

## UDPServer

UDPServer get_request 返回的是一个 (data, socket) 的 tuple，而 TCPServer 返回的是 socket  
handle 中要区分处理
`msg, sock = self.request`  
msg 已经获取，无需额外 recv

> 对于数据的传送， 你应该使用 socket 的 sendto() 和 recvfrom() 方法。 尽管传统的 send() 和 recv() 也可以达到同样的效果， 但是前面的两个方法对于 UDP 连接而言更普遍。  
> from python3-cookbook

```python
from SocketServer import BaseRequestHandler, UDPServer
import time

class TimeHandler(BaseRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        # Get message and client socket
        msg, sock = self.request
        resp = time.ctime()
        sock.sendto(resp.encode('ascii'), self.client_address)

if __name__ == '__main__':
    serv = UDPServer(('', 20000), TimeHandler)
    serv.serve_forever()

#-----------------------------
>>> from socket import socket, AF_INET, SOCK_DGRAM
>>> s = socket(AF_INET, SOCK_DGRAM)
>>> s.sendto(b'', ('localhost', 20000))
0
>>> s.recvfrom(8192)
('Thu Dec 20 10:01:01 2018', ('127.0.0.1', 20000))
```

```python

class UDPServer(TCPServer):

    """UDP server class."""

    allow_reuse_address = False

    socket_type = socket.SOCK_DGRAM

    max_packet_size = 8192

    def get_request(self):
        data, client_addr = self.socket.recvfrom(self.max_packet_size)
        return (data, self.socket), client_addr

    def server_activate(self):
        # No need to call listen() for UDP.
        pass

    def shutdown_request(self, request):
        # No need to shutdown anything.
        self.close_request(request)

    def close_request(self, request):
        # No need to close anything.
        pass
```

## ForkingMixIn

典型的 fork 使用，这里我们能看到 fork 多进程的典型使用

- 限定最大进程数，保证系统资源不至于耗尽
- 父进程 wait defunct 进程
- fork 后父进程返回
- 子进程处理请求后 `_exit()`

```python
class ForkingMixIn:

    """Mix-in class to handle each request in a new process."""

    timeout = 300
    active_children = None
    max_children = 40

    def collect_children(self):
        """Internal routine to wait for children that have exited."""
        if self.active_children is None:
            return

        while len(self.active_children) >= self.max_children:
            try:
                pid, _ = os.waitpid(-1, 0)
                self.active_children.discard(pid)
            except OSError as e:
                if e.errno == errno.ECHILD:
                    # we don't have any children, we're done
                    self.active_children.clear()
                elif e.errno != errno.EINTR:
                    break

        # Now reap all defunct children.
        for pid in self.active_children.copy():
            try:
                pid, _ = os.waitpid(pid, os.WNOHANG)
                # if the child hasn't exited yet, pid will be 0 and ignored by
                # discard() below
                self.active_children.discard(pid)
            except OSError as e:
                if e.errno == errno.ECHILD:
                    # someone else reaped it
                    self.active_children.discard(pid)

    def handle_timeout(self):
        """Wait for zombies after self.timeout seconds of inactivity.

        May be extended, do not override.
        """
        self.collect_children()

    def process_request(self, request, client_address):
        """Fork a new subprocess to process the request."""
        self.collect_children()
        pid = os.fork()
        if pid:
            # Parent process
            if self.active_children is None:
                self.active_children = set()
            self.active_children.add(pid)
            self.close_request(request) #close handle in parent process
            return
        else:
            # Child process.
            # This must never return, hence os._exit()!
            try:
                self.finish_request(request, client_address)
                self.shutdown_request(request)
                os._exit(0)
            except:
                try:
                    self.handle_error(request, client_address)
                    self.shutdown_request(request)
                finally:
                    os._exit(1)
```

## ThreadingMixIn

ThreadingMixIn 重载了 process_request 函数

1. 创建一个线程
2. 在线程中处理请求
3. 启动线程

```python

class ThreadingMixIn:
    """Mix-in class to handle each request in a new thread."""

    # Decides how threads will act upon termination of the
    # main process
    daemon_threads = False

    def process_request_thread(self, request, client_address):
        """Same as in BaseServer but as a thread.

        In addition, exception handling is done here.

        """
        try:
            self.finish_request(request, client_address)
            self.shutdown_request(request)
        except:
            self.handle_error(request, client_address)
            self.shutdown_request(request)

    def process_request(self, request, client_address):
        """Start a new thread to process the request."""
        t = threading.Thread(target = self.process_request_thread,
                             args = (request, client_address))
        t.daemon = self.daemon_threads
        t.start()
```

```python

class ForkingUDPServer(ForkingMixIn, UDPServer): pass
class ForkingTCPServer(ForkingMixIn, TCPServer): pass

class ThreadingUDPServer(ThreadingMixIn, UDPServer): pass
class ThreadingTCPServer(ThreadingMixIn, TCPServer): pass

if hasattr(socket, 'AF_UNIX'):

    class UnixStreamServer(TCPServer):
        address_family = socket.AF_UNIX

    class UnixDatagramServer(UDPServer):
        address_family = socket.AF_UNIX

    class ThreadingUnixStreamServer(ThreadingMixIn, UnixStreamServer): pass

    class ThreadingUnixDatagramServer(ThreadingMixIn, UnixDatagramServer): pass
```

## BaseRequestHandler

基础请求类，对外提供三个接口

1. setup()
2. handle()
3. finish()

使用时继承该类，通过 BaseServer 注册  
BaseServer.finish_request 中实例化 BaseRequestHandler 类，在 \_\_init\_\_函数调用中完成继承类重载的 handle() 接口的调用

```python
class BaseRequestHandler:

    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass
```

## StreamRequestHandler

提供文件操作接口

```python
class StreamRequestHandler(BaseRequestHandler):

    """Define self.rfile and self.wfile for stream sockets."""

    # Default buffer sizes for rfile, wfile.
    # We default rfile to buffered because otherwise it could be
    # really slow for large data (a getc() call per byte); we make
    # wfile unbuffered because (a) often after a write() we want to
    # read and we need to flush the line; (b) big writes to unbuffered
    # files are typically optimized by stdio even when big reads
    # aren't.
    rbufsize = -1
    wbufsize = 0

    # A timeout to apply to the request socket, if not None.
    timeout = None

    # Disable nagle algorithm for this socket, if True.
    # Use only when wbufsize != 0, to avoid small packets.
    disable_nagle_algorithm = False

    def setup(self):
        self.connection = self.request
        if self.timeout is not None:
            self.connection.settimeout(self.timeout)
        if self.disable_nagle_algorithm:
            self.connection.setsockopt(socket.IPPROTO_TCP,
                                       socket.TCP_NODELAY, True)
        self.rfile = self.connection.makefile('rb', self.rbufsize)
        self.wfile = self.connection.makefile('wb', self.wbufsize)

    def finish(self):
        if not self.wfile.closed:
            try:
                self.wfile.flush()
            except socket.error:
                # A final socket error may have occurred here, such as
                # the local error ECONNABORTED.
                pass
        self.wfile.close()
        self.rfile.close()

```

## DatagramRequestHandler

```python
class DatagramRequestHandler(BaseRequestHandler):

    """Define self.rfile and self.wfile for datagram sockets."""

    def setup(self):
        try:
            from cStringIO import StringIO
        except ImportError:
            from StringIO import StringIO
        self.packet, self.socket = self.request
        self.rfile = StringIO(self.packet)
        self.wfile = StringIO()

    def finish(self):
        self.socket.sendto(self.wfile.getvalue(), self.client_address)

```

## 版权

作者：bigfish  
许可协议：[许可协议 知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)
