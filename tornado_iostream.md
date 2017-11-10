## tornado iostream解析

封装了对socket fd底层数据的读取、写入操作，针对不同的情况实现了几种不同的读取方式。

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
        # 第一部分
        self._run_streaming_callback()
        pos = self._find_read_pos()
        if pos is not None:
            self._read_from_buffer(pos)
            return
        # 第二部分
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
        # 第三部分
        # We couldn't satisfy the read inline, so either close the stream
        # or listen for new data.
        if self.closed():
            self._maybe_run_close_callback()
        else:
            self._add_io_state(ioloop.IOLoop.READ)
   ```

* 方法可以分为三部分：
    第一部分：首先会去检测前一次是否将数据读取到了缓存，即通过self._find_read_pos()获取到的pos是否为None，如果不为None，则表示已经读取数据完成（读取到足够大小的数据：max_bytes参数、正则表达式匹配成功：regex参数，或者遇到指定的分隔符：read_until()方法中的delimiter参数），然后调用self._read_from_buffer(pos)将_read_future加入IOLoop；

    第二部分：调用self._read_to_buffer_loop()循环读取请求数据，方法中封装了第一部分的部分操作，接下来与第一部分操作一致；

    第三部分：判断该stream连接是否断开，如果已经断开调用关闭回调函数，否则将该连接再次放到IOLoop中，继续读取数据。

* tornado.iostream.BaseIOStream._find_read_pos()

   ```python
    def _find_read_pos(self):
        # 试图在读取缓冲区中找到满足当前待读取的位置
        # 如果能够满足当前读取，则返回缓冲区中的位置;如果不能，则返回None。
        if (self._read_bytes is not None and
            (self._read_buffer_size >= self._read_bytes or
             (self._read_partial and self._read_buffer_size > 0))):
            num_bytes = min(self._read_bytes, self._read_buffer_size)
            return num_bytes
        elif self._read_delimiter is not None:
            # Multi-byte delimiters (e.g. '\r\n') may straddle two
            # chunks in the read buffer, so we can't easily find them
            # without collapsing the buffer.  However, since protocols
            # using delimited reads (as opposed to reads of a known
            # length) tend to be "line" oriented, the delimiter is likely
            # to be in the first few chunks.  Merge the buffer gradually
            # since large merges are relatively expensive and get undone in
            # _consume().
            if self._read_buffer:
                loc = self._read_buffer.find(self._read_delimiter,
                                             self._read_buffer_pos)
                if loc != -1:
                    loc -= self._read_buffer_pos
                    delimiter_len = len(self._read_delimiter)
                    self._check_max_bytes(self._read_delimiter,
                                          loc + delimiter_len)
                    return loc + delimiter_len
                self._check_max_bytes(self._read_delimiter,
                                      self._read_buffer_size)
        elif self._read_regex is not None:
            # self._read_buffer在_read_to_buffer方法中定义
            if self._read_buffer:
                # 在已读取的数据中从上一次的位置开始匹配正则，self._read_buffer_pos在
                # _consume方法中定义
                m = self._read_regex.search(self._read_buffer,
                                            self._read_buffer_pos)
                # 匹配成功
                if m is not None:
                    # 获取本次读取的数据大小
                    loc = m.end() - self._read_buffer_pos
                    # 判断本次读取的数据是否超过了最大读取数据量，如果是则
                    # 报错UnsatisfiableReadError
                    self._check_max_bytes(self._read_regex, loc)
                    return loc
                # 如果没有匹配到正则表达式，则检查读取的数据量是否超过了最大读取数据量
                # self._read_buffer_size在_read_to_buffer方法中定义
                self._check_max_bytes(self._read_regex, self._read_buffer_size)
        return None
   ```

在read_until_regex()方法中，self._read_regex是存在的，self._read_delimiter是针对read_until()方法的操作。在此方法中可知，传入read_until_regex()的max_bytes必须要大于self.read_chunk_size（socket.recv每次循环接收的数据量），否则会直接报错。

* tornado.iostream.BaseIOStream._read_to_buffer_loop()

   ```python
    def _read_to_buffer_loop(self):
        # This method is called from _handle_read and _try_inline_read.
        try:
            # 根据不同的方法调用获取对应的目标数据量
            if self._read_bytes is not None:
                target_bytes = self._read_bytes
            elif self._read_max_bytes is not None:
                target_bytes = self._read_max_bytes
            elif self.reading():
                # 对于没有max_bytes参数的read_until或者read_until_close，应在扫描分
                # 隔符之前尽可能多地进行读取。
                target_bytes = None
            else:
                target_bytes = 0
            next_find_pos = 0
            
            # 假装有一个挂起的回调，以便_read_to_buffer中的EOF不会触发立即关闭回调。 
            # 在这个方法（_try_inline_read）的最后，我们要么通过_read_from_buffer
            # 建立一个真正的等待回调，要么运行关闭回调。避免程序中途退出。因为
            # 在_maybe_run_close_callback、_maybe_add_error_listener等方法中都
            # 会比较self._pending_callbacks参数值，如果该值不为0，则表示该loop过程
            # 还在执行，不能去运行关闭回调。最后的finally块中会递减该值。
            self._pending_callbacks += 1
            while not self.closed():
                # 从socket中读数据，直到得到EWOULDBLOCK（当一个非阻塞的操作没有数据
                # 操作时，如读操作时，缓存区空了，此时没有数据读了，写操作时，缓存区已
                # 满，无法再写入）或者类似的错误。在Windows上为EWOULDBLOCK，Linux
                # 上为EAGAIN
                if self._read_to_buffer() == 0:
                    break

                self._run_streaming_callback()

                # 如果已经读完了所有可以使用的字节，就跳出这个循环。不能在
                # 这里调用read_from_buffer，因为它与pending_callback和
                # error_listener机制的微妙交互。
                # 如果已经达到target_bytes，则已经读取完成了。
                if (target_bytes is not None and
                        self._read_buffer_size >= target_bytes):
                    break

                # 否则，需要调用更加昂贵的find_read_pos。 在每次读取时这样做
                # 效率不高，所以在第一次读取以及每当读取缓冲区大小加倍时都要这样做。
                if self._read_buffer_size >= next_find_pos:
                    pos = self._find_read_pos()
                    if pos is not None:
                        return pos
                    next_find_pos = self._read_buffer_size * 2
            return self._find_read_pos()
        finally:
            # 递减该属性值，以便能执行关闭回调等操作
            self._pending_callbacks -= 1
   ```

读取底层socket fd中的数据，将其保存到缓存中。通过判断该socket连接是否断开进行循环读取操作，最主要的就是self._read_to_buffer()方法，封装了对socket中读取到的数据处理过程。最终会调用socket原生的读取函数socket.recv(buffer)，即read_from_fd()函数，此函数在IOStream中被实现。

* tornado.iostream.BaseIOStream._read_from_buffer()

   ```python
    def _read_from_buffer(self, pos):
        # 尝试从缓冲区中完成当前正在等待的读取

        # 重置参数
        self._read_bytes = self._read_delimiter = self._read_regex = None
        self._read_partial = False
        self._run_read_callback(pos, False)
   ```

* tornado.iostream.BaseIOStream._run_read_callback()

   ```python
    def _run_read_callback(self, size, streaming):
        # 判断是否要运行stream回调函数，read_until_regex没有该函数
        if streaming:
            callback = self._streaming_callback
        else:
            callback = self._read_callback
            self._read_callback = self._streaming_callback = None
            # 如果self._read_future不为空
            if self._read_future is not None:
                assert callback is None
                future = self._read_future
                self._read_future = None
                future.set_result(self._consume(size))
        if callback is not None:
            assert (self._read_future is None) or streaming
            self._run_callback(callback, self._consume(size))
        else:
            # If we scheduled a callback, we will add the error listener
            # afterwards.  If we didn't, we have to do it now.
            self._maybe_add_error_listener()
   ```

首先通过streaming参数，决定callback是何值。streaming参数针对read_bytes、read_until_close这两个读取方法，而read_until、read_until_regex没有streaming_callback。针对没有streaming_callback的方法，会判断self._read_callback和self._read_future，如果self._read_future不会空，则会使用Future对象将数据返回，否则通过callback回调返回。Future详解参考：[tornado_concurrent.md](./tornado_concurrent.md)

* tornado.iostream.BaseIOStream._consume()

   ```python
    def _consume(self, loc):
        # 消耗缓存区的loc数量数据，并将其返回
        if loc == 0:
            return b""
        assert loc <= self._read_buffer_size
        # 获取已读取到缓存区的总数据self._read_buffer，并截取其中从位置
        # self._read_buffer_pos（最开始为0）到self._read_buffer_pos + loc
        # 中的数据
        b = (memoryview(self._read_buffer)
             [self._read_buffer_pos:self._read_buffer_pos + loc]
             ).tobytes()
        self._read_buffer_pos += loc
        self._read_buffer_size -= loc
        # Amortized O(1) shrink
        # (this heuristic is implemented natively in Python 3.4+
        #  but is replicated here for Python 2)
        if self._read_buffer_pos > self._read_buffer_size:
            del self._read_buffer[:self._read_buffer_pos]
            self._read_buffer_pos = 0
        return b
   ```
