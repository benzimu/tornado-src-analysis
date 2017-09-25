## tornado netutil

* tornado.netutil.bind_sockets()

该方法创建绑定到给定端口和地址的监听套接字socket。返回套接字对象的列表，比如给定的address参数映射到多个IP地址，则返回多个socket，最常见的是混合使用IPv4与IPv6，则会创建对应的两个socket。

   ```python
    def bind_sockets(port, address=None, family=socket.AF_UNSPEC,
                 backlog=_DEFAULT_BACKLOG, flags=None, reuse_port=False):

        # 当设置了端口复用时，会检查系统是否支持端口复用功能
        if reuse_port and not hasattr(socket, "SO_REUSEPORT"):
            raise ValueError("the platform doesn't support SO_REUSEPORT")

        sockets = []
        if address == "":
            address = None
        # 如果系统不支持IPv6并且参数family为AF_UNSPEC，则family选择IPv4协议
        if not socket.has_ipv6 and family == socket.AF_UNSPEC:
            family = socket.AF_INET
        if flags is None:
            flags = socket.AI_PASSIVE
        bound_port = None
        # 循环遍历获取的地址信息
        for res in set(socket.getaddrinfo(address, port, family, socket.SOCK_STREAM,
                                          0, flags)):
            af, socktype, proto, canonname, sockaddr = res
            #　排除另类数据
            if (sys.platform == 'darwin' and address == 'localhost' and
                    af == socket.AF_INET6 and sockaddr[3] != 0):
                continue

            try:
                # 创建socket对象
                sock = socket.socket(af, socktype, proto)
            except socket.error as e:
                if errno_from_exception(e) == errno.EAFNOSUPPORT:
                    continue
                raise
            # 设置close-on-exec标志位
            set_close_exec(sock.fileno())
            # 设置SO_REUSEADDR
            if os.name != 'nt':
                sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            #　设置SO_REUSEPORT
            if reuse_port:
                sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
            if af == socket.AF_INET6:
                # 在linux上，ipv6 socket也默认接受ipv4，但是这样就无法绑定到ipv4
                # 中的0.0.0.0和ipv6中的::。 
                # 在其他系统上，单独的套接字必须用于监听ipv4和ipv6。 
                # 为了保持一致性，请务必在ipv6套接字上禁用ipv4，并在需要时使用
                # 单独的ipv4套接字。

                # Windows上的Python 2.x没有IPPROTO_IPV6。
                if hasattr(socket, "IPPROTO_IPV6"):
                    sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)

            # 当port参数为None时，会自动分配绑定端口，该端口应同时绑定在IPv4和IPv6。
            # 当第一次循环时，bound_port会记录系统自动分配的port，然后应用到接下来的循环
            host, requested_port = sockaddr[:2]
            if requested_port == 0 and bound_port is not None:
                sockaddr = tuple([host, bound_port] + list(sockaddr[2:]))
            
            # 设置socket为非阻塞
            sock.setblocking(0)
            # 绑定socket
            sock.bind(sockaddr)
            # 记录下本次绑定的端口
            bound_port = sock.getsockname()[1]
            # 开启socket监听
            sock.listen(backlog)
            sockets.append(sock)
        return sockets
   ```

如上基本为socket的常规操作，主要操作放在了IPv4与IPv6的兼容方面。最终返回分别基于IPv4与IPv6两个socket对象组成的list。

* tornado.netutil.add_accept_handler()

正是该方法将web服务器与IOLoop连接了起来，主要用于添加一个IOLoop事件处理器来接受服务器socket上的新连接（来自客户端的连接）。当一个客户端连接被accept，callback函数将会被调用。

   ```python
    def add_accept_handler(sock, callback, io_loop=None):
    
        if io_loop is None:
            io_loop = IOLoop.current()

        def accept_handler(fd, events):
            # 在我们处理回调时可能会有更多的连接; 为了防止其他任务的饥饿，我们必须限制
            # 我们一次接受的连接数。理想情况下，我们将接受输入此方法时等待的连接数，
            # 但是此信息不可用（而且在运行任何回调之前重新排列此方法以调用accept()
            # 多次可能会对多进程配置中的负载平衡产生不利影响）。 相反，我们使用（默认）
            # listen backlog作为我们可以合理接受的连接数的粗略启发式。
            for i in xrange(_DEFAULT_BACKLOG):
                try:
                    connection, address = sock.accept()
                except socket.error as e:
                    # _ERRNO_WOULDBLOCK表示我们接受了每个有用的连接
                    if errno_from_exception(e) in _ERRNO_WOULDBLOCK:
                        return
                    # ECONNABORTED表示有个链接已经被关闭了但任然在接受队列中
                    if errno_from_exception(e) == errno.ECONNABORTED:
                        continue
                    raise
                # 调用callback
                callback(connection, address)
        # 将服务器端sock以读事件注册到epoll，即epoll一直监听着服务器端socket，只要有
        # 客户端连接到服务器端socket，epoll就会触发，IOLoop就会调用accept_handler方
        # 法，最终调用callback(connection, address)
        io_loop.add_handler(sock, accept_handler, IOLoop.READ)
   ```

