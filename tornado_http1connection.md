## tornado http1connection解析

tornado http1connection主要是对http协议进行了封装。

* tornado.http1connection.HTTP1ServerConnection.start_serving()

   ```python
    def start_serving(self, delegate):
        # 这里断言delegate是httputil.HTTPServerConnectionDelegate的实例
        assert isinstance(delegate, httputil.HTTPServerConnectionDelegate)
        self._serving_future = self._server_request_loop(delegate)
        self.stream.io_loop.add_future(self._serving_future,
                                       lambda f: f.result())
   ```

开始处理这个连接上的请求，在tornado.HTTPServer.handle_stream()中调用此方法，传入的参数为HTTPServer的实例，即delegate为HTTPServer实例，由于HTTPServer继承至httputil.HTTPServerConnectionDelegate，所以断言成功，程序开始执行self._server_request_loop()。

* tornado.http1connection.HTTP1ServerConnection._server_request_loop()

   ```python
    @gen.coroutine
    def _server_request_loop(self, delegate):
        try:
            while True:
                # 初始化HTTP1Connection实例
                conn = HTTP1Connection(self.stream, False,
                                       self.params, self.context)
                # 调用delegate的start_request处理连接请求
                request_delegate = delegate.start_request(self, conn)
                try:
                    # 读取http响应
                    ret = yield conn.read_response(request_delegate)
                except (iostream.StreamClosedError,
                        iostream.UnsatisfiableReadError):
                    return
                except _QuietException:
                    # This exception was already logged.
                    conn.close()
                    return
                except Exception:
                    gen_log.error("Uncaught exception", exc_info=True)
                    conn.close()
                    return
                # 如果ret为false，则表示读取了完整的http响应
                if not ret:
                    return
                yield gen.moment
        finally:
            delegate.on_close(self)v
   ```

方法中调用了delegate.start_request(self, conn)，即[tornado.httpserver.HTTPServer.start_request()](./tornado_httpserver.md/#start_request)