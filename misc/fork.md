## SocketServer.ForkingMixIn

典型的 fork 使用

- 限定最大进程数
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

        # If we're above the max number of children, wait and reap them until
        # we go back below threshold. Note that we use waitpid(-1) below to be
        # able to collect children in size(<defunct children>) syscalls instead
        # of size(<children>): the downside is that this might reap children
        # which we didn't spawn, which is why we only resort to this when we're
        # above max_children.

        # 如果达到最大的进程数，则等待直到某个进程退出
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
        # 收集所有的僵尸进程
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
            # 父进程关闭 request 并退出
            self.close_request(request) #close handle in parent process
            return
        else:
            # Child process.
            # This must never return, hence os._exit()!
            # 子线程处理 request,之后调用 os._exit() 退出
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
