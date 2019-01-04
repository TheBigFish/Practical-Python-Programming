## 原文

[A Web Crawler With asyncio Coroutines](http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html)

## 解析

###使用生成器构建协程

### 精简版

Fetcher 函数中 yield 一个 Future 对象，select 注册连接事件 on_connected，on_connected 会调用该 Future 对象的 set_result 方法。

step 函数通过调用生成器的 send 方法推动生成器的函数执行。获取到 Fetcher yield 出来的 Future 时，给该 Future 注册一个回调，该回调为 step 本身。

select 连接事件产生后，on_connected 被调用，继而调用 set_result，继而调用 step 注册的函数即 step 本身，继而调用 send 方法，推动 Fetcher 前进。

select 取消对连接事件的监听，生成器结束，step 捕获 StopIteration 异常，返回

协程 Fetcher.fetch 执行结束

异步的程序通过 yield 得以顺序执行，而不会阻塞主线程

外部时间调用 send 方法推动协程执行

> The task starts the fetch generator by sending None into it. Then fetch runs until it yields a future, which the task captures as next_future. When the socket is connected, the event loop runs the callback on_connected, which resolves the future, which calls step, which resumes fetch.

```python
import socket
from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ

selector = DefaultSelector()


def read(sock):
    f = Future()

    def on_readable():
        f.set_result(sock.recv(4096))

    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f  # Read one chunk.
    selector.unregister(sock.fileno())
    return chunk


def read_all(sock):
    response = []
    # Read whole response.
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)

    return b''.join(response)

class Future:
    def __init__(self):
        self.result = None
        self._callbacks = []

    def add_done_callback(self, fn):
        self._callbacks.append(fn)

    def set_result(self, result):
        self.result = result
        for fn in self._callbacks:
            fn(self)

class Fetcher:
    def __init__(self, url):
        self.response = b''  # Empty array of bytes.
        self.url = url
        self.sock = None

    def connected(self, key, mask):
        print('connected!')

    def fetch(self):
        print("fetch generator start")
        sock = socket.socket()
        sock.setblocking(False)
        try:
            sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass

        f = Future()

        def on_connected():
            print("on_connected call back to push fetch")
            f.set_result(None)

        print("register on_connected")
        selector.register(sock.fileno(),
                          EVENT_WRITE,
                          on_connected)
        print("Future() yield {}".format(f))
        yield f
        selector.unregister(sock.fileno())
        print('connected!')

class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()
        f.set_result(None)
        self.step(f)

    def step(self, future):
        print("step in")
        try:
            print("call send() to push generator")
            next_future = self.coro.send(future.result)
            print("Future() get{}".format(next_future))
        except StopIteration:
            print("step stop")
            return

        print("add call back to future")
        next_future.add_done_callback(self.step)
        print("step out")

def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()

# Begin fetching http://xkcd.com/353/
fetcher = Fetcher('/353/')
Task(fetcher.fetch())

loop()


# step in
# call send() to push generator
# fetch generator start
# register on_connected
# Future() yield <__main__.Future object at 0x7f0beadb6898>
# Future() get<__main__.Future object at 0x7f0beadb6898>
# add call back to future
# step out
# on_connected call back to push fetch
# step in
# call send() to push generator
# connected!
# step stop
```

### 使用 yield from 代理协程

修改协程 fetch，socket 连接后 发送 http 请求同时获取结果

1. select 监听到可读事件
2. 调用注册的 on_readable
3. 同时调用注册的 Future.set_result 函数
4. 执行 f.set_result(sock.recv(4096))
5. 执行 temp_result = sock.recv(4096)
6. 执行 self.result = temp_result
7. 执行 fn(self)
8. 执行 next_future = self.coro.send(future.result)，把读取的数据传回协程
9. chunk 获取读取的数据

```python
    def fetch(self):
        sock = socket.socket()
        sock.setblocking(False)
        try:
            sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass

        f = Future()

        def on_connected():
            f.set_result(None)

        selector.register(sock.fileno(),
                          EVENT_WRITE,
                          on_connected)
        yield f
        selector.unregister(sock.fileno())
        print('connected!')

        request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(self.url)
        sock.send(request.encode('ascii'))

        while True:
            f = Future()

            def on_readable():
                f.set_result(sock.recv(4096))

            selector.register(sock.fileno(),
                              EVENT_READ,
                              on_readable)
            chunk = yield f
            selector.unregister(sock.fileno())
            if chunk:
                self.response += chunk
            else:
                # Done reading.
                break
        print(self.response)
```

建立一个读协程

select 注册读事件 -> 读取数据 -> select 取消读事件

整个过程顺序执行，自然流畅

```python
def read(sock):
    f = Future()

    def on_readable():
        f.set_result(sock.recv(4096))

    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f  # Read one chunk.
    selector.unregister(sock.fileno())
    return chunk
```

读取所有数据 read_all

```python
def read_all(sock):
    response = []
    # Read whole response.
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)

    return b''.join(response)
```

```python
class Fetcher:
def fetch(self):
        # ... connection logic from above, then:
    sock.send(request.encode('ascii'))
    self.response = yield from read_all(sock)
```

程序有一个瑕疵，使用 yield 等待 future，而使用 yield from 代理 yield 协程。这样就必须根据等待的内容区分是使用 yield 还是 yield from。

能不能够统一使用 yield from，回答是可以。

只需要给 Future 增加一个 函数 \_\_iter\_\_

```python
# Method on Future class.
def __iter__(self):
    # Tell Task to resume me here.
    yield self
    return self.result
```

此时 yield from f 等同于 yield f

> We take advantage of the deep correspondence in Python between generators and iterators. Advancing a generator is, to the caller, the same as advancing an iterator.

> yield from iterator

Future 增加 \_\_iter\_\_ 变成可迭代对象类，对 \_\_iter\_\_ 的调用会产生一个生成器，程序会在 yield self 处挂起，Task send 会传入 result，chunk = yielf f 获取到结果。而
此时 return self.result 同样会返回结果给外部，比如 chunk = yield from f。

这样，我们便可以使用 yield from 取代 yield。

### yield from 过程

由下面过程可见，yield from 第一步是生成迭代器 \_i = iter(EXPR)

```python
RESULT = yield from EXPR

_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:
                try:
                    _y = _m(*_x)
                except StopIteration as _e:

                    _r = _e.value
                    break
        else:
            try:
                if _s is None:
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:
                _r = _e.value
                break
RESULT = _r
```
