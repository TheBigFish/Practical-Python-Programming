# ioloop

A level-triggered I/O loop for non-blocking sockets.  
条件触发、非阻塞的 socket I/O 循环

一个简单的 TCP 服务：

## contents

- [ioloop](#ioloop)
  - [contents](#contents)
  - [head](#head)
  - [IOLoop](#ioloop)
  - [IOLoop.\_\_init\_\_](#ioloopinit)
  - [IOLoop.instance](#ioloopinstance)
    - [IOLoop.add_handler](#ioloopaddhandler)
    - [IOLoop.update_handler](#ioloopupdatehandler)
    - [IOLoop.start](#ioloopstart)
  - [\_Select](#select)
  - [\_poll](#poll)
  - [signal](#signal)

```python
import errno
import functools
from tornado import ioloop
import socket


def connection_ready(sock, fd, events):
    while True:
        try:
            connection, address = sock.accept()
        except socket.error, e:
            if e.args[0] not in (errno.EWOULDBLOCK, errno.EAGAIN):
                raise
            return
        connection.setblocking(0)
        #handle_connection(connection, address)


sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setblocking(0)
sock.bind(("", 8080))
sock.listen(128)

io_loop = ioloop.IOLoop.instance()
callback = functools.partial(connection_ready, sock)
io_loop.add_handler(sock.fileno(), callback, io_loop.READ)
io_loop.start()
```

## head

```python


import bisect
import errno
import os
import logging
import select
import time
import traceback

from tornado import stack_context

try:
    import signal
except ImportError:
    signal = None

try:
    import fcntl
except ImportError:
    if os.name == 'nt':
        from tornado import win32_support
        from tornado import win32_support as fcntl
    else:
        raise
```

## IOLoop

```python
class IOLoop(object):
    # Constants from the epoll module
    _EPOLLIN = 0x001
    _EPOLLPRI = 0x002
    _EPOLLOUT = 0x004
    _EPOLLERR = 0x008
    _EPOLLHUP = 0x010
    _EPOLLRDHUP = 0x2000
    _EPOLLONESHOT = (1 << 30)
    _EPOLLET = (1 << 31)

    # Our events map exactly to the epoll events
    NONE = 0
    READ = _EPOLLIN
    WRITE = _EPOLLOUT
    ERROR = _EPOLLERR | _EPOLLHUP | _EPOLLRDHUP
```

## IOLoop.\_\_init\_\_

\_running: 控制循环退出
\_handlers: 事件对象，参数为 (fd, events)

```python
    def __init__(self, impl=None):
        self._impl = impl or _poll()
        if hasattr(self._impl, 'fileno'):
            self._set_close_exec(self._impl.fileno())
        self._handlers = {}
        self._events = {}
        self._callbacks = []
        self._timeouts = []
        self._running = False
        self._stopped = False
        self._blocking_signal_threshold = None

        # Create a pipe that we send bogus data to when we want to wake
        # the I/O loop when it is idle
        if os.name != 'nt':
            r, w = os.pipe()
            self._set_nonblocking(r)
            self._set_nonblocking(w)
            self._set_close_exec(r)
            self._set_close_exec(w)
            self._waker_reader = os.fdopen(r, "rb", 0)
            self._waker_writer = os.fdopen(w, "wb", 0)
        else:
            self._waker_reader = self._waker_writer = win32_support.Pipe()
            r = self._waker_writer.reader_fd
        self.add_handler(r, self._read_waker, self.READ)
        raise
```

## IOLoop.instance

单例模式，返回一个全局唯一的 IOLoop 对象

```python
    @classmethod
    def instance(cls):
        """Returns a global IOLoop instance.

        Most single-threaded applications have a single, global IOLoop.
        Use this method instead of passing around IOLoop instances
        throughout your code.

        A common pattern for classes that depend on IOLoops is to use
        a default argument to enable programs with multiple IOLoops
        but not require the argument for simpler applications:

            class MyClass(object):
                def __init__(self, io_loop=None):
                    self.io_loop = io_loop or IOLoop.instance()
        """
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance

    @classmethod
    def initialized(cls):
        return hasattr(cls, "_instance")
```

### IOLoop.add_handler

注册 socket 事件回调

1. 将回调事件使用上下文包裹后加入事件处理列表
2. 向 poll 注册 socket 事件

```python
    def add_handler(self, fd, handler, events):
        """Registers the given handler to receive the given events for fd."""
        self._handlers[fd] = stack_context.wrap(handler)
        self._impl.register(fd, events | self.ERROR)
```

### IOLoop.update_handler

```python
    def update_handler(self, fd, events):
        """Changes the events we listen for fd."""
        self._impl.modify(fd, events | self.ERROR)

    def remove_handler(self, fd):
        """Stop listening for events on fd."""
        self._handlers.pop(fd, None)
        self._events.pop(fd, None)
        try:
            self._impl.unregister(fd)
        except (OSError, IOError):
            logging.debug("Error deleting fd from IOLoop", exc_info=True)

    def set_blocking_signal_threshold(self, seconds, action):
        """Sends a signal if the ioloop is blocked for more than s seconds.

        Pass seconds=None to disable.  Requires python 2.6 on a unixy
        platform.

        The action parameter is a python signal handler.  Read the
        documentation for the python 'signal' module for more information.
        If action is None, the process will be killed if it is blocked for
        too long.
        """
        if not hasattr(signal, "setitimer"):
            logging.error("set_blocking_signal_threshold requires a signal module "
                       "with the setitimer method")
            return
        self._blocking_signal_threshold = seconds
        if seconds is not None:
            signal.signal(signal.SIGALRM,
                          action if action is not None else signal.SIG_DFL)

    def set_blocking_log_threshold(self, seconds):
        """Logs a stack trace if the ioloop is blocked for more than s seconds.
        Equivalent to set_blocking_signal_threshold(seconds, self.log_stack)
        """
        self.set_blocking_signal_threshold(seconds, self.log_stack)

    def log_stack(self, signal, frame):
        """Signal handler to log the stack trace of the current thread.

        For use with set_blocking_signal_threshold.
        """
        logging.warning('IOLoop blocked for %f seconds in\n%s',
                        self._blocking_signal_threshold,
                        ''.join(traceback.format_stack(frame)))
```

### IOLoop.start

启动 I/O 事件循环

1. 调用注册的回调对象
2. 调用注册的延时回调对象
3. poll
4. 调用注册的 socket 事件回调对象

```python
    def start(self):
        """Starts the I/O loop.

        The loop will run until one of the I/O handlers calls stop(), which
        will make the loop stop after the current event iteration completes.
        """
        if self._stopped:
            self._stopped = False
            return
        self._running = True
        while True:
            # Never use an infinite timeout here - it can stall epoll
            poll_timeout = 0.2

            # Prevent IO event starvation by delaying new callbacks
            # to the next iteration of the event loop.

            # 调用注册的回调对象
            callbacks = self._callbacks
            self._callbacks = []
            for callback in callbacks:
                self._run_callback(callback)

            if self._callbacks:
                poll_timeout = 0.0

            # 调用注册的延时回调对象
            if self._timeouts:
                now = time.time()
                while self._timeouts and self._timeouts[0].deadline <= now:
                    timeout = self._timeouts.pop(0)
                    self._run_callback(timeout.callback)
                if self._timeouts:
                    milliseconds = self._timeouts[0].deadline - now
                    poll_timeout = min(milliseconds, poll_timeout)

            # 如果外部设置，则退出事件循环
            if not self._running:
                break

            # TODO
            if self._blocking_signal_threshold is not None:
                # clear alarm so it doesn't fire while poll is waiting for
                # events.
                signal.setitimer(signal.ITIMER_REAL, 0, 0)

            try:
                event_pairs = self._impl.poll(poll_timeout)
            except Exception, e:
                # Depending on python version and IOLoop implementation,
                # different exception types may be thrown and there are
                # two ways EINTR might be signaled:
                # * e.errno == errno.EINTR
                # * e.args is like (errno.EINTR, 'Interrupted system call')
                if (getattr(e, 'errno', None) == errno.EINTR or
                    (isinstance(getattr(e, 'args', None), tuple) and
                     len(e.args) == 2 and e.args[0] == errno.EINTR)):
                    continue
                else:
                    raise

            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL,
                                 self._blocking_signal_threshold, 0)

            # Pop one fd at a time from the set of pending fds and run
            # its handler. Since that handler may perform actions on
            # other file descriptors, there may be reentrant calls to
            # this IOLoop that update self._events
            self._events.update(event_pairs)
            while self._events:
                fd, events = self._events.popitem()
                try:
                    self._handlers[fd](fd, events)
                except (KeyboardInterrupt, SystemExit):
                    raise
                except (OSError, IOError), e:
                    if e.args[0] == errno.EPIPE:
                        # Happens when the client closes the connection
                        pass
                    else:
                        logging.error("Exception in I/O handler for fd %d",
                                      fd, exc_info=True)
                except:
                    logging.error("Exception in I/O handler for fd %d",
                                  fd, exc_info=True)
        # reset the stopped flag so another start/stop pair can be issued
        self._stopped = False
        if self._blocking_signal_threshold is not None:
            signal.setitimer(signal.ITIMER_REAL, 0, 0)

    def stop(self):
        """Stop the loop after the current event loop iteration is complete.
        If the event loop is not currently running, the next call to start()
        will return immediately.

        To use asynchronous methods from otherwise-synchronous code (such as
        unit tests), you can start and stop the event loop like this:
          ioloop = IOLoop()
          async_method(ioloop=ioloop, callback=ioloop.stop)
          ioloop.start()
        ioloop.start() will return after async_method has run its callback,
        whether that callback was invoked before or after ioloop.start.
        """
        self._running = False
        self._stopped = True
        self._wake()

    def running(self):
        """Returns true if this IOLoop is currently running."""
        return self._running

    def add_timeout(self, deadline, callback):
        """Calls the given callback at the time deadline from the I/O loop.

        Returns a handle that may be passed to remove_timeout to cancel.
        """
        timeout = _Timeout(deadline, stack_context.wrap(callback))
        bisect.insort(self._timeouts, timeout)
        return timeout

    def remove_timeout(self, timeout):
        """Cancels a pending timeout.

        The argument is a handle as returned by add_timeout.
        """
        self._timeouts.remove(timeout)

    def add_callback(self, callback):
        """Calls the given callback on the next I/O loop iteration.

        It is safe to call this method from any thread at any time.
        Note that this is the *only* method in IOLoop that makes this
        guarantee; all other interaction with the IOLoop must be done
        from that IOLoop's thread.  add_callback() may be used to transfer
        control from other threads to the IOLoop's thread.
        """
        self._callbacks.append(stack_context.wrap(callback))
        self._wake()

    def _wake(self):
        try:
            self._waker_writer.write("x")
        except IOError:
            pass

    def _run_callback(self, callback):
        try:
            callback()
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            self.handle_callback_exception(callback)

    def handle_callback_exception(self, callback):
        """This method is called whenever a callback run by the IOLoop
        throws an exception.

        By default simply logs the exception as an error.  Subclasses
        may override this method to customize reporting of exceptions.

        The exception itself is not passed explicitly, but is available
        in sys.exc_info.
        """
        logging.error("Exception in callback %r", callback, exc_info=True)

    def _read_waker(self, fd, events):
        try:
            while True:
                self._waker_reader.read()
        except IOError:
            pass

    def _set_nonblocking(self, fd):
        flags = fcntl.fcntl(fd, fcntl.F_GETFL)
        fcntl.fcntl(fd, fcntl.F_SETFL, flags | os.O_NONBLOCK)

    def _set_close_exec(self, fd):
        flags = fcntl.fcntl(fd, fcntl.F_GETFD)
        fcntl.fcntl(fd, fcntl.F_SETFD, flags | fcntl.FD_CLOEXEC)


class _Timeout(object):
    """An IOLoop timeout, a UNIX timestamp and a callback"""

    # Reduce memory overhead when there are lots of pending callbacks
    __slots__ = ['deadline', 'callback']

    def __init__(self, deadline, callback):
        self.deadline = deadline
        self.callback = callback

    def __cmp__(self, other):
        return cmp((self.deadline, id(self.callback)),
                   (other.deadline, id(other.callback)))


class PeriodicCallback(object):
    """Schedules the given callback to be called periodically.

    The callback is called every callback_time milliseconds.
    """
    def __init__(self, callback, callback_time, io_loop=None):
        self.callback = callback
        self.callback_time = callback_time
        self.io_loop = io_loop or IOLoop.instance()
        self._running = False

    def start(self):
        self._running = True
        timeout = time.time() + self.callback_time / 1000.0
        self.io_loop.add_timeout(timeout, self._run)

    def stop(self):
        self._running = False

    def _run(self):
        if not self._running: return
        try:
            self.callback()
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            logging.error("Error in periodic callback", exc_info=True)
        if self._running:
            self.start()


class _EPoll(object):
    """An epoll-based event loop using our C module for Python 2.5 systems"""
    _EPOLL_CTL_ADD = 1
    _EPOLL_CTL_DEL = 2
    _EPOLL_CTL_MOD = 3

    def __init__(self):
        self._epoll_fd = epoll.epoll_create()

    def fileno(self):
        return self._epoll_fd

    def register(self, fd, events):
        epoll.epoll_ctl(self._epoll_fd, self._EPOLL_CTL_ADD, fd, events)

    def modify(self, fd, events):
        epoll.epoll_ctl(self._epoll_fd, self._EPOLL_CTL_MOD, fd, events)

    def unregister(self, fd):
        epoll.epoll_ctl(self._epoll_fd, self._EPOLL_CTL_DEL, fd, 0)

    def poll(self, timeout):
        return epoll.epoll_wait(self._epoll_fd, int(timeout * 1000))


class _KQueue(object):
    """A kqueue-based event loop for BSD/Mac systems."""
    def __init__(self):
        self._kqueue = select.kqueue()
        self._active = {}

    def fileno(self):
        return self._kqueue.fileno()

    def register(self, fd, events):
        self._control(fd, events, select.KQ_EV_ADD)
        self._active[fd] = events

    def modify(self, fd, events):
        self.unregister(fd)
        self.register(fd, events)

    def unregister(self, fd):
        events = self._active.pop(fd)
        self._control(fd, events, select.KQ_EV_DELETE)

    def _control(self, fd, events, flags):
        kevents = []
        if events & IOLoop.WRITE:
            kevents.append(select.kevent(
                    fd, filter=select.KQ_FILTER_WRITE, flags=flags))
        if events & IOLoop.READ or not kevents:
            # Always read when there is not a write
            kevents.append(select.kevent(
                    fd, filter=select.KQ_FILTER_READ, flags=flags))
        # Even though control() takes a list, it seems to return EINVAL
        # on Mac OS X (10.6) when there is more than one event in the list.
        for kevent in kevents:
            self._kqueue.control([kevent], 0)

    def poll(self, timeout):
        kevents = self._kqueue.control(None, 1000, timeout)
        events = {}
        for kevent in kevents:
            fd = kevent.ident
            flags = 0
            if kevent.filter == select.KQ_FILTER_READ:
                events[fd] = events.get(fd, 0) | IOLoop.READ
            if kevent.filter == select.KQ_FILTER_WRITE:
                events[fd] = events.get(fd, 0) | IOLoop.WRITE
            if kevent.flags & select.KQ_EV_ERROR:
                events[fd] = events.get(fd, 0) | IOLoop.ERROR
        return events.items()
```

## \_Select

将 select 包装，提供 poll 接口

```python

class _Select(object):
    """A simple, select()-based IOLoop implementation for non-Linux systems"""
    def __init__(self):
        self.read_fds = set()
        self.write_fds = set()
        self.error_fds = set()
        self.fd_sets = (self.read_fds, self.write_fds, self.error_fds)

    def register(self, fd, events):
        if events & IOLoop.READ: self.read_fds.add(fd)
        if events & IOLoop.WRITE: self.write_fds.add(fd)
        if events & IOLoop.ERROR:
            self.error_fds.add(fd)
            # Closed connections are reported as errors by epoll and kqueue,
            # but as zero-byte reads by select, so when errors are requested
            # we need to listen for both read and error.
            self.read_fds.add(fd)

    def modify(self, fd, events):
        self.unregister(fd)
        self.register(fd, events)

    def unregister(self, fd):
        self.read_fds.discard(fd)
        self.write_fds.discard(fd)
        self.error_fds.discard(fd)

    def poll(self, timeout):
        readable, writeable, errors = select.select(
            self.read_fds, self.write_fds, self.error_fds, timeout)
        events = {}
        for fd in readable:
            events[fd] = events.get(fd, 0) | IOLoop.READ
        for fd in writeable:
            events[fd] = events.get(fd, 0) | IOLoop.WRITE
        for fd in errors:
            events[fd] = events.get(fd, 0) | IOLoop.ERROR
        return events.items()

```

## \_poll

选择一个全局的 poll 实现  
优先级 epoll > kqueue > select

```python
# Choose a poll implementation. Use epoll if it is available, fall back to
# select() for non-Linux platforms
if hasattr(select, "epoll"):
    # Python 2.6+ on Linux
    _poll = select.epoll
elif hasattr(select, "kqueue"):
    # Python 2.6+ on BSD or Mac
    _poll = _KQueue
else:
    try:
        # Linux systems with our C module installed
        import epoll
        _poll = _EPoll
    except:
        # All other systems
        import sys
        if "linux" in sys.platform:
            logging.warning("epoll module not found; using select()")
        _poll = _Select

```

## signal

```python
import time
import signal

def catcher(signum, _):
    print "beat!"

signal.signal(signal.SIGALRM, catcher)
signal.setitimer(signal.ITIMER_REAL, 2, 2)

while True:
    time.sleep(5)

# delivers a "beat!" message every 2 seconds.
```
