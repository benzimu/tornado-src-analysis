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

    * tornado.ioloop.IOLoop.current()

   ```python
    @staticmethod
    def current(instance=True):
        current = getattr(IOLoop._current, "instance", None)
        if current is None and instance:
            return IOLoop.instance()
        return current
   ```

该函数只有四行，它先判断当前线程中是否有IOLoop实例正在运行或者被IOLoop.make_current()标记过，如果结果为真就直接返回当前IOLoop，否则调用IOLoop.instance()去创建IOLoop实例。

    * tornado.ioloop.IOLoop.instance()

```python
    @staticmethod
    def instance():
        # 第一层检查
        if not hasattr(IOLoop, "_instance"):
            with IOLoop._instance_lock:
                # 第二层检查
                if not hasattr(IOLoop, "_instance"):
                    # New instance after double check
                    IOLoop._instance = IOLoop()
        return IOLoop._instance
```

在该方法中使用了两次判断，用以实现单例。第一层判断只是为了提升性能，其实只需第二层判断就已经可以实现单例模式。但是如果没有第一层判断，我们只有实例化IOLoop的时候需要加锁，其他时候IOLoop实例已经存在了，不需要加锁了，但是这时其他需要IOLoop实例的线程还是必须等待锁，锁的存在明显降低了效率，有性能损耗。

两层检查都没通过时，会初始化IOLoop对象（调用IOLoop._instance = IOLoop()）。此时，调用IOLoop父类[tornado.util.Configurable](./tornado_util_configurable.md)的__new__()方法实例化IOLoop对象。因为没有调用tornado.util.Configurable.configure()函数配置实现类，因此Configurable会调用tornado.ioloop.IOLoop.configurable_default()默认配置实现类（不同系统选择不同的IO多路复用机制，即epoll、kqueue、select）。

    * tornado.ioloop.IOLoop.configurable_default()

   ```python
    @classmethod
    def configurable_default(cls):
        # 判断系统是否实现epoll，如果有就使用EPollIOLoop作为实现类
        if hasattr(select, "epoll"):
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        # 判断系统是否实现kqueue， 如果有就使用KQueueIOLoop作为实现类
        if hasattr(select, "kqueue"):
            # Python 2.6+ on BSD or Mac
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        # 否则使用SelectIOLoop作为实现类
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
   ```

此处我们默认选择EPollIOLoop作为实现类。通过Configurable会生成EPollIOLoop实例，并调用tornado.platform.epoll.EPollIOLoop.initialize()。

    * tornado.platform.epoll.EPollIOLoop

   ```python
    class EPollIOLoop(PollIOLoop):
        def initialize(self, **kwargs):
            # 调用PollIOLoop.initialize()
            super(EPollIOLoop, self).initialize(impl=select.epoll(), **kwargs)
   ```

    * tornado.ioloop.PollIOLoop.initialize()

   ```python
        def initialize(self, impl, time_func=None, **kwargs):
        super(PollIOLoop, self).initialize(**kwargs)
        self._impl = impl
        if hasattr(self._impl, 'fileno'):
            set_close_exec(self._impl.fileno())
        self.time_func = time_func or time.time
        self._handlers = {}
        self._events = {}
        self._callbacks = collections.deque()
        self._timeouts = []
        self._cancellations = 0
        self._running = False
        self._stopped = False
        self._closing = False
        self._thread_ident = None
        self._blocking_signal_threshold = None
        self._timeout_counter = itertools.count()

        # Create a pipe that we send bogus data to when we want to wake
        # the I/O loop when it is idle
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
   ```