## tornado ioloop

tornado ioloop是一个基于水平触发的非阻塞socket的IO事件循环。在Linux系统上会使用epoll，Mac和BSD系统中使用kqueue，否则使用select。在分析源码之前需要搞清楚几个知识点：

* [socket（TCP三次握手、四次挥手）](./socket.md)
* [IO多路复用（select、epoll、kqueue）](./io_multiplexing.md)

ioloop.IOLoop，继承至tornado.util.Configurable（主要用于子类的创建）。IOLoop首先会获取一个全局锁，以保证全局只有一个IOLoop，即单例模式。IOLoop使用事例：

   ```python
    import errno
    import functools
    import tornado.ioloop
    import socket

    def connection_ready(sock, fd, events):
        while True:
            try:
                connection, address = sock.accept()
            except socket.error as e:
                if e.args[0] not in (errno.EWOULDBLOCK, errno.EAGAIN):
                    raise
                return
            connection.setblocking(0)
            handle_connection(connection, address)

    if __name__ == '__main__':
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.setblocking(0)
        sock.bind(("", port))
        sock.listen(128)
        
        # 获取IOLoop对象
        io_loop = tornado.ioloop.IOLoop.current()
        callback = functools.partial(connection_ready, sock)
        io_loop.add_handler(sock.fileno(), callback, io_loop.READ)
        # 启动IOLoop
        io_loop.start()
   ```

从事例中知道，使用ioloop事件循环，首先需要获取IOLoop对象，即调用tornado.ioloop.IOLoop.current()：

   ```python
    @staticmethod
    def current(instance=True):
        current = getattr(IOLoop._current, "instance", None)
        if current is None and instance:
            return IOLoop.instance()
        return current
   ```

该函数只有四行，它先判断当前线程中是否有IOLoop实例正在运行或者被IOLoop.make_current()标记过，如果结果为真就直接返回当前IOLoop，否则调用IOLoop.instance()去创建IOLoop实例。

```python
    @staticmethod
    def instance():
        if not hasattr(IOLoop, "_instance"):
            with IOLoop._instance_lock:
                if not hasattr(IOLoop, "_instance"):
                    # New instance after double check
                    IOLoop._instance = IOLoop()
        return IOLoop._instance
```

在该方法中使用了两次判断，用以实现单例。第一层判断只是为了提升性能，其实只需第二层判断就已经可以实现单例模式。但是如果没有第一层判断，我们只有实例化IOLoop的时候需要加锁，其他时候IOLoop实例已经存在了，不需要加锁了，但是这时其他需要IOLoop实例的线程还是必须等待锁，锁的存在明显降低了效率，有性能损耗。
