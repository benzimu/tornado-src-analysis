## tornado定时器PeriodicCallback

tornado.ioloop.PeriodicCallback是tornado实现的定时器。

   ```python
    class Application(tornado.web.Application):
        def __init__(self):
            handlers = [
                (r"/", HomeHandler),
            ]
            settings = dict(
                debug=True,
            )
            super(Application, self).__init__(handlers, **settings)
   ```

当创建tornado Application时，如果设置“debug=True”，tornado会在源程序修改后自动编译，而不需要我们手动重启。

* tornado.web.Application.__init__()

   ```python
    def __init__(self, handlers=None, default_host=None, transforms=None,
                 **settings):
        #####省略#####
        
        # 当设置了debug=True时，tornado会默认设置autoreload=True
        if self.settings.get('debug'):
            self.settings.setdefault('autoreload', True)
            self.settings.setdefault('compiled_template_cache', False)
            self.settings.setdefault('static_hash_cache', False)
            self.settings.setdefault('serve_traceback', True)

        #####省略#####

        # 判断是否设置了autoreload，如果设置了就会调用autoreload.start()，
        # 启动自动重载功能
        if self.settings.get('autoreload'):
            from tornado import autoreload
            autoreload.start()
   ```

* tornado.autoreload.start()

   ```python
    def start(io_loop=None, check_time=500):
        # 获取IOLoop实例
        io_loop = io_loop or ioloop.IOLoop.current()
        # 如果当前IOLoop实例是弱引用的则直接返回
        if io_loop in _io_loops:
            return
        _io_loops[io_loop] = True
        if len(_io_loops) > 1:
            gen_log.warning("tornado.autoreload started more than once in the same process")
        modify_times = {}
        # 生成回调函数
        callback = functools.partial(_reload_on_update, modify_times)
        # 初始化tornado.PeriodicCallback定时器
        scheduler = ioloop.PeriodicCallback(callback, check_time, io_loop=io_loop)
        # 开始执行定时器
        scheduler.start()
   ```

* tornado.ioloop.PeriodicCallback.start()

   ```python
    def start(self):
        # 设置定时器运行中
        self._running = True
        # 初始化下次的超时事件的最后期限
        self._next_timeout = self.io_loop.time()
        # 关键方法，对下次超时事件的封装
        self._schedule_next()
   ```

* tornado.ioloop.PeriodicCallback._schedule_next()

   ```python
    def _schedule_next(self):
        if self._running:
            # 获取当前时间
            current_time = self.io_loop.time()
            # 如果当前时间已经超过了超时事件的最后期限，则重新设置超时时间
            if self._next_timeout <= current_time:
                # self.callback_time为autoreload.start()方法中初始化定时器时传入的
                # check_time，即500毫秒
                callback_time_sec = self.callback_time / 1000.0
                self._next_timeout += (math.floor((current_time - self._next_timeout) /
                                                  callback_time_sec) + 1) * callback_time_sec
            # 关键所在，添加超时事件，将self._run作为超时后的回调函数
            self._timeout = self.io_loop.add_timeout(self._next_timeout, self._run)
   ```

* tornado.ioloop.PeriodicCallback._run()

   ```python
    def _run(self):
        # 判断定时器是否还在运行
        if not self._running:
            return
        try:
            # 调用autoreload.start()方法中初始化时传入的超时回调函数
            return self.callback()
        except Exception:
            self.io_loop.handle_callback_exception(self.callback)
        finally:
            # 无论如何，都会再次调用self._schedule_next()再次添加超时事件到IOLoop中，
            # 这样就会一直循环，即定时器操作
            self._schedule_next()
   ```

在PeriodicCallback._schedule_next()的最后一行执行的添加超时事件就会被IOLoop下次循环中。

* <div id="add_timeout"></div>tornado.ioloop.IOLoop.add_timeout()

   ```python
    def add_timeout(self, deadline, callback, *args, **kwargs):
        # 判断超时事件的最后期限deadline是否为实数，一般为实数
        if isinstance(deadline, numbers.Real):
            return self.call_at(deadline, callback, *args, **kwargs)
        # 在使用call_later()方法设置超时事件时deadline为datetime.timedelta类型
        elif isinstance(deadline, datetime.timedelta):
            return self.call_at(self.time() + timedelta_to_seconds(deadline),
                                callback, *args, **kwargs)
        else:
            raise TypeError("Unsupported deadline %r" % deadline)
   ```

tornado规定继承至IOLoop的子类必须实现add_timeout()或者call_at()方法，因为默认实现只是相互调用，而没有实质作用。tornado.ioloop.PollIOLoop则实现了call_at()。

* tornado.ioloop.PollIOLoop.call_at()

   ```python
    def call_at(self, deadline, callback, *args, **kwargs):
        # 初始化_Timeout对象，该对象是对超时事件的封装，同时重写了__lt__()、__le__()
        # 两个方法，实现了_Timeout的大小比较。
        timeout = _Timeout(
            deadline,
            functools.partial(stack_context.wrap(callback), *args, **kwargs),
            self)
        # 通过堆排序添加timeout到self._timeouts列表中，因此确定了self._timeouts[0]
        # 总是最小的，即最后期限deadline最小的，当最后期限相同时则为最先添加
        # 到self._timeouts的
        heapq.heappush(self._timeouts, timeout)
        return timeout
   ```

执行该函数之后timeouts就会被添加到self._timeouts中，当tornado IOLoop的epoll.poll()函数再次醒来时，则会重新迭代，然后调用self._timeouts，进行相关判断处理。详解参考：[tornado_ioloop.md](./tornado_ioloop.md/#start)
