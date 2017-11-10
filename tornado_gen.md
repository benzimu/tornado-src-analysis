## tornado gen解析

tornado.gen是一个基于生成器的接口，可以更容易地在异步环境中工作。使用gen模块的代码在技术上是异步的，但它被写成一个单独的生成器，而不是另外的函数集合。

* <div id="with_timeout"></div>tornado.gen.with_timeout()

   ```python
    def with_timeout(timeout, future, io_loop=None, quiet_exceptions=()):
        # 在超时时间内封装Future对象
        # 如果传入的future在超时之前没有完成，则引发TimeoutError，可以通过
        # .IOLoop.add_timeout（即datetime.timedelta或者相对于.IOLoop.time的绝对
        # 时间）允许的任何形式来指定。
        # 如果封装的Future在超时之后失败，则将记录该异常，除非它是“quiet_exceptions”
        # （可能是一个异常类型或一系列类型）中包含的类型。

        # 将一个yielded对象转换为一个Future
        future = convert_yielded(future)
        # 初始化一个新的Future对象
        result = Future()
        # 将新的Future对象与待处理的future关联，用于超时处理、结果处理
        chain_future(future, result)
        if io_loop is None:
            io_loop = IOLoop.current()
        
        # future超时处理函数
        def error_callback(future):
            try:
                future.result()
            except Exception as e:
                if not isinstance(e, quiet_exceptions):
                    app_log.error("Exception in Future %r after timeout",
                                  future, exc_info=True)
        # IOLoop超时回调函数
        def timeout_callback():
            result.set_exception(TimeoutError("Timeout"))
            # In case the wrapped future goes on to fail, log it.
            future.add_done_callback(error_callback)
        # 添加超时事件到IOLoop
        timeout_handle = io_loop.add_timeout(
            timeout, timeout_callback)
        # 根据future不同类型，分别处理删除IOLoop中future的超时回调事件
        if isinstance(future, Future):
            # We know this future will resolve on the IOLoop, so we don't
            # need the extra thread-safety of IOLoop.add_future (and we also
            # don't care about StackContext here.
            future.add_done_callback(
                lambda future: io_loop.remove_timeout(timeout_handle))
        else:
            # concurrent.futures.Futures may resolve on any thread, so we
            # need to route them back to the IOLoop.
            io_loop.add_future(
                future, lambda future: io_loop.remove_timeout(timeout_handle))
        return result
   ```

首先会初始化一个Future对象，用于获取结果，即result。然后调用chain_future将future与result对象绑定，参考详解：[tornado_concurrent.md](./tornado_concurrent.md/#chain_future)。接着会注册超时事件到IOLoop，具体参考详解：[tornado_ioloop_PeriodicCallback.md](./tornado_ioloop_PeriodicCallback.md/#add_timeout)。如果timeout超时了，则IOLoop会调用timeout_callback，然后result调用set_exception抛出异常，并终止Future，即设置self._done为True；future也同样抛出异常，会将异常写入日志中。如果在timeout超时之前future操作成功完成，则会将其返回数据写到result中，并删除注册到IOLoop中的超时回调事件。