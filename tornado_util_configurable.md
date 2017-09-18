## tornado util configurable

tornado.util.Configurable，一个配置类，是工厂模式的实现，通过使用构造函数（__new__()）作为工厂方法。其子类必须实现configurable_base()、configurable_default()、initialize()。通过调用configure()函数去配置当基类（不是指Configurable，而是继承至Configurable的类，如tornado.ioloop.IOLoop）被实例化时使用的实现类，以及配置其实现类初始化的关键字参数。

示例：

   ```python
    from tornado import httpclient

    httpclient.AsyncHTTPClient.configure\
    ("tornado.curl_httpclient.CurlAsyncHTTPClient", max_clients=10000)

    http_client = httpclient.AsyncHTTPClient()
   ```

以tornado.httpclient.AsyncHTTPClient为示例来开始tornado.util.Configurable的剖析。从tornado源码可知，AsyncHTTPClient继承至Configurable，同时tornado.curl_httpclient.CurlAsyncHTTPClient继承至AsyncHTTPClient。

* 第二行AsyncHTTPClient调用configure去设置它的实现类及关键字参数max_clients=10000。其源码中直接调用了父类（Configurable）的configure()函数。

    * tornado.httpclient.AsyncHTTPClient.configure()

   ```python
    @classmethod
    def configure(cls, impl, **kwargs):
        super(AsyncHTTPClient, cls).configure(impl, **kwargs)
   ```

    * tornado.util.Configurable.configure()

   ```python
    @classmethod
    def configure(cls, impl, **kwargs):
        # cls为AsyncHTTPClient，获取可配置层次结构的基类base（AsyncHTTPClient）
        base = cls.configurable_base()
        # 由上面的例子得：impl="tornado.curl_httpclient.CurlAsyncHTTPClient"
        if isinstance(impl, (str, unicode_type)):
            # 引入tornado.curl_httpclient.CurlAsyncHTTPClient到当前上下文环境
            impl = import_object(impl)
        if impl is not None and not issubclass(impl, cls):
            raise ValueError("Invalid subclass of %s" % cls)
        # 通过全局变量保存数据，这两个变量是初始化实例
        # （tornado.util.Configurable.__new__()）时非常重要的数据
        # 值为：tornado.curl_httpclient.CurlAsyncHTTPClient
        base.__impl_class = impl 
        # 值为：{"max_clients": 10000}
        base.__impl_kwargs = kwargs 
   ```

* 第三行获取AsyncHTTPClient实例，将会调用tornado.util.Configurable.__new__()函数。

    * tornado.util.Configurable.__new__()

   ```python
    def __new__(cls, *args, **kwargs):
        # cls为AsyncHTTPClient，获取可配置层次结构的基类，
        # 通常是其自身（如tornado.httpclient.AsyncHTTPClient.configurable_base()）
        base = cls.configurable_base()
        init_kwargs = {}
        # 判断cls是否是基类base
        if cls is base:
            # 获取当前配置的实现类，因为之前配置过实现类，即第二行，
            # 所以得到impl为tornado.curl_httpclient.CurlAsyncHTTPClient
            impl = cls.configured_class()
            # 判断configure()函数配置的关键字参数是否为空
            if base.__impl_kwargs:
                # 更新初始化参数字典，因为之前配置过关键字参数，即第二行，
                # base.__impl_kwargs={"max_clients": 10000}
                init_kwargs.update(base.__impl_kwargs)
        else:
            # 实现类即为cls
            impl = cls
        # 更新初始化参数字典
        init_kwargs.update(kwargs)
        # 实例化cls，如示例，instance为tornado.curl_httpclient.CurlAsyncHTTPClient
        instance = super(Configurable, cls).__new__(impl)
        # 初始化实例参数
        instance.initialize(*args, **init_kwargs)
        return instance
   ```

    * tornado.util.Configurable.configured_class()

   ```python
    @classmethod
    def configured_class(cls):
        # cls为AsyncHTTPClient
        base = cls.configurable_base()
        # 判断有没有调用tornado.util.Configurable.configure()函数进行配置，
        # 如果没有配置过，就调用默认设置configurable_default()
        if cls.__impl_class is None:
            base.__impl_class = cls.configurable_default()
        return base.__impl_class
   ```

tornado.util.Configurable.configured_class()函数是选取实现类的关键，它会判断是否调用过tornado.util.Configurable.configure()函数去配置实现类了，然后以此选择相应的实现类。

