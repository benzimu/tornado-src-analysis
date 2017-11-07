## tornado iostream解析

* <div id="read_until_regex"></div>tornado.iostream.BaseIOStream.read_until_regex()

   ```python
    def read_until_regex(self, regex, callback=None, max_bytes=None):
        # 设置读取完成后的回调函数
        future = self._set_read_callback(callback)
        # 编译正则表达式，并保存到self._read_regex
        self._read_regex = re.compile(regex)
        # 保存最大读取数据长度到self._read_max_bytes
        self._read_max_bytes = max_bytes
        try:
            self._try_inline_read()
        except UnsatisfiableReadError as e:
            # Handle this the same way as in _handle_events.
            gen_log.info("Unsatisfiable read, closing connection: %s" % e)
            self.close(exc_info=True)
            return future
        except:
            if future is not None:
                # Ensure that the future doesn't log an error because its
                # failure was never examined.
                future.add_done_callback(lambda f: f.exception())
            raise
        return future
   ```

通过正则表达式匹配客户端发来的数据，http协议规定了数据结构，每行数据后面都会有一个CRLF，而请求行与请求头数据结尾也会有一个CRLF，即请求头数据后面有两个CRLF，请求体也是如此。CRLF即为回车（Carriage-Return，CR，\r）、换行（Line-Feed，LF,\n）。Windows下CRLF表示：\r\n；Unix下CRLF表示：\n。tornado.http1connection.HTTP1Connection._read_message()方法调用本方法传入的regex参数为"\r?\n\r?\n"，刚好兼容了Windows和Unix。

* tornado.iostream.BaseIOStream._set_read_callback()

   ```python
    def _set_read_callback(self, callback):
        assert self._read_callback is None, "Already reading"
        assert self._read_future is None, "Already reading"
        if callback is not None:
            self._read_callback = stack_context.wrap(callback)
        else:
            self._read_future = TracebackFuture()
        return self._read_future
   ```

通过read_until_regex方法读取请求数据时，是没有callback的，即callback为None，所以_set_read_callback会返回一个Future对象实例，用于返回异步执行结果。

* tornado.iostream.BaseIOStream._try_inline_read()

   ```python
    def _try_inline_read(self):
        # 尝试从缓存数据中完成当前读操作。
        # 如果此次读操作能在无阻塞的情况下完成，则在下次IOLoop迭代中执行读回调函数，
        # 否则为此次读事件在套接字上启动监听

        # 查看是否从前一次的读操作中获取到了数据
        self._run_streaming_callback()
        pos = self._find_read_pos()
        if pos is not None:
            self._read_from_buffer(pos)
            return
        self._check_closed()
        try:
            pos = self._read_to_buffer_loop()
        except Exception:
            # If there was an in _read_to_buffer, we called close() already,
            # but couldn't run the close callback because of _pending_callbacks.
            # Before we escape from this function, run the close callback if
            # applicable.
            self._maybe_run_close_callback()
            raise
        if pos is not None:
            self._read_from_buffer(pos)
            return
        # We couldn't satisfy the read inline, so either close the stream
        # or listen for new data.
        if self.closed():
            self._maybe_run_close_callback()
        else:
            self._add_io_state(ioloop.IOLoop.READ)
   ```