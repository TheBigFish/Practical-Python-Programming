```python
import asyncore
import socket
import datetime


class TimeChannel(asyncore.dispatcher):

    def handle_write(self):
        self.send(datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"))

    def handle_close(self):
        self.close()


class TimeServer(asyncore.dispatcher):

    def __init__(self, port=37):
        asyncore.dispatcher.__init__(self)
        self.port = port
        self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
        self.bind(("", port))
        self.listen(5)
        print "listening on port", self.port

    def handle_accept(self):
        channel, _ = self.accept()
        TimeChannel(channel)


server = TimeServer(8037)
print server
asyncore.loop()
```

simple client

```python

import asyncore, socket


class Client(asyncore.dispatcher_with_send):
    def __init__(self, host, port, message):
        asyncore.dispatcher.__init__(self)
        self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
        self.connect((host, port))
        self.out_buffer = message

    def handle_close(self):
        self.close()

    def handle_read(self):
        s = self.recv(1024)
        print 'Received', s
        self.close()


c = Client('127.0.0.1', 8037, 'Hello, world')
asyncore.loop()
```
