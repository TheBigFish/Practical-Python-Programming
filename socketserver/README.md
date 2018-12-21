# SocketServer

创建网络服务器的同步框架

## Introduction

同步 socket 库(服务端阻塞直到请求完成)  
用于 TCP, UDP, Unix streams, and Unix datagrams 服务
使用 mix-in 模式支持多线程/多进程处理请求

框架分为 server(服务器)类和 request handler(请求处理)类

## StreamRequestHandler

提供一个类似文件操作的接口
wfile 用于写，rfile 用于读

```python
from socketserver import StreamRequestHandler, TCPServer

class EchoHandler(StreamRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        # self.rfile is a file-like object for reading
        for line in self.rfile:
            # self.wfile is a file-like object for writing
            self.wfile.write(line)

if __name__ == '__main__':
    serv = TCPServer(('', 20000), EchoHandler)
    serv.serve_forever()
```
