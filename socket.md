## socket:

1. socket是网络进程之间的通讯方式，是对TCP/IP协议的封装。socket并不是像HTTP、TCP、IP一样的协议，而是一组调用接口（API）。通过socket，可以使用TCP/IP协议，即可以在网络上传输数据。
2. http是应用层协议，web开发中最常见的协议。当我们在浏览器输入一个网址，比如http://www.google.com时:
    * 浏览器首先会去查看本地hosts文件，通过域名获取其对应的IP地址，如果本地没有找到，则继续向上层请求DNS服务器，DNS服务器就像一个树结构，一层一层的向上递进。
    * 获取IP地址之后，浏览器会根据IP与默认端口80，通过TCP协议三次握手与服务器建立socket连接。
    * 在linux下，一切皆文件。系统将每一个socket连接也抽象成一个文件，客户端与服务器建立连接之后，各自在本地进程中维护一个“文件”，然后可以向自己文件写入内容供对方读取或者读取对方内容，通讯结束时关闭文件。

## TCP三次握手、四次挥手

* **TCP连接到断开过程：**

    <div align=center><img src="static/tcp.jpg" alignwidth="700" height="700" alt="tcp连接过程"/></div>

    * 三次握手：
    
        第一次握手：客户端尝试连接服务器，向服务器发送syn包（同步序列编号Synchronize Sequence Numbers），syn=j，客户端进入SYN_SEND状态等待服务器确认;

        第二次握手：服务器接收客户端syn包并确认（ack=j+1），同时向客户端发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态;

        第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1），此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手

    * 四次挥手
    
        第一次挥手：主机1（可以使客户端，也可以是服务器端），设置Sequence Number和Acknowledgment Number，向主机2发送一个FIN报文段；此时，主机1进入FIN_WAIT_1状态；这表示主机1没有数据要发送给主机2了；

        第二次挥手：主机2收到了主机1发送的FIN报文段，向主机1回一个ACK报文段，Acknowledgment Number为Sequence Number加1；主机1进入FIN_WAIT_2状态；主机2告诉主机1，我“同意”你的关闭请求；
        
        第三次挥手：主机2向主机1发送FIN报文段，请求关闭连接，同时主机2进入LAST_ACK状态；
        
        第四次挥手：主机1收到主机2发送的FIN报文段，向主机2发送ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了

* **client状态过程：**
   
   <div align=center><img src="static/client-status.png" alignwidth="600" height="600" alt="tcp连接过程"/></div>

* **server状态过程：**
   
   <div align=center><img src="static/server-status.png" alignwidth="600" height="600" alt="tcp连接过程"/></div>

## socket实例

   * server.py

   ```python
    import socket
    import StringIO
    
    HOST = "127.0.0.1"
    PORT = 3267
    
    # 创建一个IPV4且基于TCP协议的socket对象
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 绑定监听端口
    server.bind((HOST, PORT))
    # 开始监听端口，并且等待连接的最大数量为5
    server.listen(5)

    print "waiting for connection..."

    while True:
        # 接受连接（此方法阻塞）
        conn, addr = server.accept()
        print "Connected by ", addr
        
        buffer = StringIO.StringIO()
        while True:
            # 每次最多读取1k数据
            data = conn.recv(1024)
            if data:
                print "receive client data: ", data
                buffer.write(data)
                conn.sendall("Hello, {}".format(data))
            else: 
                break
            
        print "receive client ALL datas: ", buffer.getvalue()
        buffer.close()
        conn.close()
        print 'Connection from %s:%s closed.' % addr
   ```

   * client.py
   
   ```python
    import socket
    
    HOST = "127.0.0.1"
    PORT = 3267

    # 创建一个IPV4且基于TCP协议的socket对象
    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 建立连接
    client.connect((HOST, PORT))

    while True:
        # 获取用户控制台输入
        data = raw_input("please input something: ")
        client.send(data)

        result = client.recv(1024)
        print "client result: ", result

    client.close()
   ```



