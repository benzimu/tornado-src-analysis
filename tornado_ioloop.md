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

该函数只有四行，它先判断当前进程中是否有IOLoop实例正在运行或者被IOLoop.make_current()标记过，如果结果为真就直接返回当前IOLoop，否则调用IOLoop.instance()去创建IOLoop实例。

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
            # 设置select.epoll描述符在子进程执行exec()族函数时自动关掉
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
        # 创建一个pipe（管道），当IOLoop处于空闲，即无读写事件时，
        # 会向该pipe中发送虚假数据从而叫醒IOLoop
        self._waker = Waker()
        # 将唤醒者（waker）读文件描述符注册到epoll，
        # 当另一线程调用tornado.ioloop.PollIOLoop.add_callback()时，
        # 会调用self._waker.wake()唤醒IOLoop，从而让注册的回调函数运行。
        # 由于唤醒正在轮训中的IOLoop比它自动唤醒（超时）消耗资源相对昂贵，
        # 所以在IOLoop同一线程中不会去主动唤醒它。
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
   ```

方法中都是进行一些全局数据的初始化工作。其中对epoll文件描述符设置fork时自动关闭以及叫醒IOLoop机制可参考[tornado_platform_posix.md](./tornado_platform_posix.md)

IOLoop初始化完成之后就会调用IOLoop.start()方法去启动IOLoop，是IOLoop的核心。此方法在PollIOLoop中实现。

* <div id="start"></div>tornado.ioloop.PollIOLoop.start()

   ```python
        def start(self):
        if self._running:
            raise RuntimeError("IOLoop is already running")
        self._setup_logging()
        if self._stopped:
            self._stopped = False
            return
        old_current = getattr(IOLoop._current, "instance", None)
        IOLoop._current.instance = self
        self._thread_ident = thread.get_ident()
        self._running = True
        
        # signal.set_wakeup_fd解决了事件循环中的条件竞争：select/poll/etc在开始进入
        # 中断休眠之前一个信号可能会到达，因此信号可能在没有唤醒select时被处理消耗掉。
        # 解决方法与C语言中的同步机制一样，将信号处理程序写入管道，然后通过select获取。
        old_wakeup_fd = None
        if hasattr(signal, 'set_wakeup_fd') and os.name == 'posix':
            try:
                # 将唤醒者写管道文件描述符注册为wakeup_fd，当一个信号到来
                # 时（如SIGINT，即Ctrl+C），会向其中写入"\0"，从而唤醒select或poll。
                # 该函数返回上次设置的文件描述符，如果之前没设置过，则返回-1
                old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                # 当返回值为-1则说明之前已经设置过wakeup_fd，然后重置
                if old_wakeup_fd != -1:
                    signal.set_wakeup_fd(old_wakeup_fd)
                    old_wakeup_fd = None
            except ValueError:
                old_wakeup_fd = None

        try:
            while True:
                # 为了避免IO事件饥饿，将新添加的回调延迟到事件循环的下一次迭代中。
                
                # epoll水平触发模式下，当有大量文件描述符就绪需要处理时，
                # 可能会导致事件太多而没有执行到新添加的回调，这样就会造成IO事件饥饿。
                ncallbacks = len(self._callbacks)

                # timeouts是tornado封装好的超时处理器
                due_timeouts = [] # 保存本次迭代需要执行的超时任务
                if self._timeouts:
                    now = self.time()
                    while self._timeouts:
                        # 超时事件的回调函数已经被取消，即该超时事件已无效
                        if self._timeouts[0].callback is None:
                            heapq.heappop(self._timeouts)
                            self._cancellations -= 1
                        # 当前时间是否已超过超时事件的最后期限，如已超过，则将其取出并
                        # 保存到due_timeouts，然后执行
                        elif self._timeouts[0].deadline <= now:
                            due_timeouts.append(heapq.heappop(self._timeouts))
                        else:
                            break
                    # 优化，当超时事件被取消的次数大于512次并且大于超时事件数量的一
                    # 半时，清理所有被取消的超时事件
                    if (self._cancellations > 512 and
                            self._cancellations > (len(self._timeouts) >> 1)):
                        self._cancellations = 0
                        self._timeouts = [x for x in self._timeouts
                                          if x.callback is not None]
                        heapq.heapify(self._timeouts)

                # 执行所有callback函数，及已经超时的超时事件
                for i in range(ncallbacks):
                    self._run_callback(self._callbacks.popleft())
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback)
                # 释放资源
                due_timeouts = timeout = None

                # 如果有回调函数，则epoll.poll(timeout)函数的timeout为0
                if self._callbacks:
                    poll_timeout = 0.0
                # 如果没有回调函数且有超时事件，则poll的timeout为：
                # 首先获取最近的超时事件的最后期限与当前事件的差值，然后该值与poll的
                # timeout的默认值两者取较小值，再与0比较取较大值
                elif self._timeouts:
                    poll_timeout = self._timeouts[0].deadline - self.time()
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                else:
                    poll_timeout = _POLL_TIMEOUT

                if not self._running:
                    break

                # 取消信号定时器
                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)

                try:
                    # 整个方法的核心，即epoll.poll(timeout)
                    event_pairs = self._impl.poll(poll_timeout)
                except Exception as e:
                    if errno_from_exception(e) == errno.EINTR:
                        continue
                    else:
                        raise

                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL,
                                     self._blocking_signal_threshold, 0)

                # 处理IO事件
                self._events.update(event_pairs)
                while self._events:
                    fd, events = self._events.popitem()
                    try:
                        # 通过文件描述符获取在PollIOLoop.add_handler()
                        # 方法中绑定到self._handlers中的socket对象及处理函数
                        fd_obj, handler_func = self._handlers[fd]
                        # 调用处理器函数
                        handler_func(fd_obj, events)
                    except (OSError, IOError) as e:
                        if errno_from_exception(e) == errno.EPIPE:
                            pass
                        else:
                            # 如果有异常则调用异常处理函数
                            self.handle_callback_exception(self._handlers.get(fd))
                    except Exception:
                        self.handle_callback_exception(self._handlers.get(fd))
                # 释放资源
                fd_obj = handler_func = None

        finally:
            self._stopped = False
            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL, 0, 0)
            IOLoop._current.instance = old_current
            if old_wakeup_fd is not None:
                signal.set_wakeup_fd(old_wakeup_fd)
   ```

对start()方法中的timeouts的详解可以参考[tornado_ioloop_PeriodicCallback.md](./tornado_ioloop_PeriodicCallback.md)，其中针对特定实例做了相关分析，能加深理解。callbacks与timeouts的处理是相似的。

tornado IOLoop中对epoll.register()、epoll.modify()、epoll.unregister()分别做了封装，对应PollIOLoop.add_handler()、PollIOLoop.update_handler()、PollIOLoop.remove_handler()，分别表示注册文件描述符到epoll中、更新epoll中监听文件描述符的事件类型、删除epoll中监听的文件描述符。




