# 一、app.run() 在做什么？

执行 `app.run()` 便启动了 Flask 服务，这个服务为什么能够监听 http 请求并做出响应？让我们进入 `run` 函数内部一探究竟。

```python{.line-numbers}
def run(self, host='localhost', port=5000, **options):
    from werkzeug import run_simple
    if 'debug' in options:
        self.debug = options.pop('debug')
    options.setdefault('use_reloader', self.debug)
    options.setdefault('use_debugger', self.debug)
    return run_simple(host, port, self, **options)
```

可以看到，`run` 函数3-6行做了些参数默认值设置，最后将参数传入 `run_simple` 并调用返回，注意，第3个参数是 `Flask` 对象（留意 `Flask` 对象的传递）。`run_simple` 是从 `werkzeug` 导入的。

```python{.line-numbers}
def run_simple(hostname, port, application, use_reloader=False,
               use_debugger=False, use_evalex=True,
               extra_files=None, reloader_interval=1, threaded=False,
               processes=1, request_handler=None, static_files=None,
               passthrough_errors=False, ssl_context=None):

    ...

    def inner():
        make_server(hostname, port, application, threaded,
                    processes, request_handler,
                    passthrough_errors, ssl_context).serve_forever()

    ...

    if use_reloader:
        test_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        test_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        test_socket.bind((hostname, port))
        test_socket.close()
        run_with_reloader(inner, extra_files, reloader_interval)
    else:
        inner()
```

`run_simple()` 最终调用 `inner()`，这个函数做了两件事：

1. 构造一个服务：`make_server()`
2. 启动服务：`.serve_forever()`

```python{.line-numbers}
def make_server(host, port, app=None, threaded=False, processes=1,
                request_handler=None, passthrough_errors=False,
                ssl_context=None):
    if threaded and processes > 1:
        raise ValueError("cannot have a multithreaded and "
                         "multi process server.")
    elif threaded:
        return ThreadedWSGIServer(host, port, app, request_handler,
                                  passthrough_errors, ssl_context)
    elif processes > 1:
        return ForkingWSGIServer(host, port, app, processes, request_handler,
                                 passthrough_errors, ssl_context)
    else:
        return BaseWSGIServer(host, port, app, request_handler,
                              passthrough_errors, ssl_context)
```

`make_server()` 返回一个 `BaseWSGIServer` 对象。

```python{.line-numbers}
class BaseWSGIServer(HTTPServer, object):
    ...

    def __init__(self, host, port, app, handler=None,
                passthrough_errors=False, ssl_context=None):
    ...

    HTTPServer.__init__(self, (host, int(port)), handler)
    self.app = app
    
    ...
```

`BaseWSGIServer` 继承 `HTTPServer`，`HTTPServer` 继承来自 Python 标准库 `SocketServer.TCPServer` 类，其最终继承自 `SocketServer.BaseServer`。注意 `Flask` 对象被赋值给 `self.app`。

`BaseServer` 类有一个 `serve_forever` 方法：

```python{.line-numbers}
def serve_forever(self, poll_interval=0.5):
    self.__is_shut_down.clear()
    try:
        while not self.__shutdown_request:
            r, w, e = _eintr_retry(select.select, [self], [], [],
                                    poll_interval)
            
            if self.__shutdown_request:
                break
            if self in r:
                self._handle_request_noblock()
    finally:
        self.__shutdown_request = False
        self.__is_shut_down.set()
```

其中有一个 `while` 循环，在不断执行:

```python{.line-numbers}
if ready:
    self._handle_request_noblock()
```

```python{.line-numbers}
def _handle_request_noblock(self):
    ...
    self.process_request(request, client_address)
    ...
```

`_handle_request_noblock()` 调用 `process_request()`，这是处理请求的函数，这个函数实例化 `self.RequestHandlerClass` 类来处理请求，这个类是 `BaseServer` 初始化时传入的参数。

到此我们知道，`app.run()` 实际上实例化了一个 `socketserver.BaseServer` 类，并调用该类的 `server_forever` 方法，这个方法主体是一个 `while` 循环，最终在不断实例化 `self.RequestHandlerClass` 来处理请求，在不中断 `while` 的情况下，程序会一直运行，不断接收请求，处理请求。

# 二、Flask 如何收到请求？

`while` 程序在不断监听请求，当接收到请求时，实例化 `self.RequestHandlerClass` 来处理请求。这个变量在 `werkzeug.BaseWSGIServer` 的 `__init__` 方法中被赋值（以下第10行）：

```python{.line-numbers}
class BaseWSGIServer(HTTPServer, object):
    multithread = False
    multiprocess = False

    def __init__(self, host, port, app, handler=None,
                 passthrough_errors=False, ssl_context=None):
        if handler is None:
            handler = WSGIRequestHandler
        self.address_family = select_ip_version(host, port)
        HTTPServer.__init__(self, (host, int(port)), handler)
```

默认值为 `werkzeug.WSGIRequestHandler`，这个类最终继承自 `SocketServer.BaseRequestHandler`，也就是说`isinstance(self.RequestHandlerClass, SocketServer.BaseRequestHandler)`。

```python{.line-numbers}
class BaseRequestHandler:

    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass
```

`SocketServer.BaseRequestHandler` 实例化时调用 `self.handle` 方法。注意，此时 `Flask` 对象存在于 `self.server.app` 中。`werkzeug.WSGIRequestHandler` 将这个方法覆写：

```python{.line-numbers}
class WSGIRequestHandler(BaseHTTPRequestHandler, object):

    def handle(self):
        try:
            return BaseHTTPRequestHandler.handle(self)
        except (socket.error, socket.timeout), e:
            self.connection_dropped(e)
        except:
            if self.server.ssl_context is None or not is_ssl_error():
                raise

    def handle_one_request(self):
        self.raw_requestline = self.rfile.readline()
        if not self.raw_requestline:
            self.close_connection = 1
        elif self.parse_request():
            return self.run_wsgi()


class BaseHTTPRequestHandler(object):

    def handle(self):
        self.close_connection = 1

        self.handle_one_request()
        while not self.close_connection:
            self.handle_one_request()
```

覆写的方法调用 `BaseHTTPRequestHandler.handle(self)`，内部调用了 `self.handle_one_request` 方法，最终调用了 `WSGIRequestHandler.run_wsgi` 方法：

```python{.line-numbers}
class WSGIRequestHandler(BaseHTTPRequestHandler, object):

    def run_wsgi(self):
        app = self.server.app
        environ = self.make_environ()

        ...

        def execute(app):
            application_iter = app(environ, start_response)
            try:
                for data in application_iter:
                    write(data)
                # make sure the headers are sent
                if not headers_sent:
                    write('')
            finally:
                if hasattr(application_iter, 'close'):
                    application_iter.close()
                application_iter = None

        ...

        try:
            execute(app)
        except (socket.error, socket.timeout), e:
            ...
```

`run_wsgi` 方法第一行即取出 `Flask` 对象，然后将其传入 `execute` 函数并调用，`execute` 第一行 `application_iter = app(environ, start_response)`，这是在调用 `Flask.__call__` 方法。

```python{.line-numbers}
class Flask(object):

    def wsgi_app(self, environ, start_response):
        with self.request_context(environ):
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
            response = self.make_response(rv)
            response = self.process_response(response)
            return response(environ, start_response)

    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```

至此，请求被传到了 `Flask` 对象中处理，或者说 `Flask` 收到了请求。

# 三、Flask 如何处理请求？

Web 服务器把 `environ, start_response` 两个参数传入 `Flask.__call__` 处理，正常处理完后将 `Flask.__call__` 返回的数据写入响应体中。Flask 处理请求其实是接收这两个参数并返回数据。

```python{.line-numbers}
class Flask(object):

    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)

    def wsgi_app(self, environ, start_response):
        with self.request_context(environ):
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
            response = self.make_response(rv)
            response = self.process_response(response)
            return response(environ, start_response)
```

`Flask.__call__` 调用 `Flask.wsgi_app` 并返回，不将处理逻辑直接实现在 `Flask.__call__` 里而封装在 `Flask.wsgi_app`里，是为了能应用中间件，例如：`app.wsgi_app = MyMiddleware(app.wsgi_app)`。

`Flask.wsgi_app` 详解：

- 第8行（`self.preprocess_request()`）：执行处理请求前的某些处理。即执行所有被 `before_request` 装饰的函数，如果返回值非空，则不进行真正的请求处理。

- 第10行（`self.dispatch_request()`）：分发处理请求。将 URL 与对应的处理函数匹配，执行处理。

- 第11行（`self.make_response()`）：将返回值转换为响应对象。

- 第12行（`self.process_response()`）：执行处理请求后的某些处理。即执行所有被 `after_request` 装饰的函数。

- 第13行（`response(environ, start_response)`）：返回一个可迭代对象。

FLask 先调用 `Flask.preprocess_request` 处理请求，再调用与 URL 对应的函数处理（注意，如果 `Flask.preprocess_request` 返回值非空，则跳过与 URL 对应的处理函数），然后把返回值转换为响应对象，最后调用 `Flask.process_response` 处理。

## 3.1 请求前置处理

```python{.line-numbers}
class Flask(object):

    def before_request(self, f):
        self.before_request_funcs.append(f)
        return f

    def preprocess_request(self):
        for func in self.before_request_funcs:
            rv = func()
            if rv is not None:
                return rv
```

`Flask.before_request` 是一个装饰器，会把被装饰的函数添加到列表 `self.before_request_funcs` 中， `self.preprocess_request` 遍历 `self.before_request_funcs`，执行所有函数，如果执行的函数返回值非空，则返回。

## 3.2 请求处理

```python{.line-numbers}
class Flask(object):

    def match_request(self):
        rv = _request_ctx_stack.top.url_adapter.match()
        request.endpoint, request.view_args = rv
        return rv

    def dispatch_request(self):
        try:
            endpoint, values = self.match_request()
            return self.view_functions[endpoint](**values)
        except HTTPException, e:
            handler = self.error_handlers.get(e.code)
            if handler is None:
                return e
            return handler(e)
        except Exception, e:
            handler = self.error_handlers.get(500)
            if self.debug or handler is None:
                raise
            return handler(e)
```

`match_request` 中调用的 `_request_ctx_stack.top.url_adapter.match()`，是 `_RequestContext.url_adapter.match()`。

```python{.line-numbers}
class _RequestContext(object):

    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
```

`_RequestContext.url_adapter` 是 Flask 对象中 `url_map.bind_to_environ(environ)` 返回的值，`Map.bin_to_environ` 返回 `MapAdapter` 对象。

`MapAdapter.match()` 会根据 URL 与 请求方法（GET、POST 等）（URL、请求方法等信息会从 `environ` 中获取）返回当前请求的 endpoint 与 参数。

```python{.line-numbers}
# eg
>>> urls.match("/downloads/42")
('downloads/show', {'id': 42})
```

在执行 URL 与 endpoint 解析前，需要先添加匹配规则。Flask 如何做的呢？

```python{.line-numbers}
from werkzeug.routing import Map, Rule


class Flask(object):

    def __init__(self, package_name):
        self.url_map = Map()
```

`Map` 存储 URL 规则和配置参数，可以通过 `Map.add` 添加 URL 匹配规则。

```python{.line-numbers}
class Flask(object):

    def add_url_rule(self, rule, endpoint, **options):
        options['endpoint'] = endpoint
        options.setdefault('methods', ('GET',))
        self.url_map.add(Rule(rule, **options))

    def route(self, rule, **options):
        def decorator(f):
            self.add_url_rule(rule, f.__name__, **options)
            self.view_functions[f.__name__] = f
            return f
        return decorator
```

方法 `Flask.add_url_rule` 与装饰器 `Flask.route` 都能添加匹配规则。装饰器 `route` 会将被装饰的函数名作为 endpoint，并将函数名作为 key，函数作为 value，存在 Flask 对象的 `self.view_functions` 属性中。

通过 `endpoint, values = self.match_request()` 解析得到 endpoint 与 请求参数，再从 `self.view_functions` 中取出对应的函数处理（`return self.view_functions[endpoint](**values)`）。

> endpoint 作用与意义可参考：https://stackoverflow.com/questions/19261833/what-is-an-endpoint-in-flask/19262349#19262349

如果处理过程中抛出异常，会根据异常码（`e.code`）取出对应的函数，执行异常处理函数。

异常处理函数通过装饰器 `Flask.errorhandler` 添加，将错误码作为 key，被装饰的函数作为 value，存入 Flask 的属性 `self.error_handlers` 中：

```python{.line-numbers}
class Flask(object):

    def errorhandler(self, code):
        def decorator(f):
            self.error_handlers[code] = f
            return f
        return decorator
```

## 3.3 生成响应对象

为什么需要这一步，直接返回 Python 内置数据类型不行吗？

不行，因为 WSGI 规定 application 端必须返回一个可迭代对象[1]。

```python{.line-numbers}
class Flask(object):

    def make_response(self, rv):
        if isinstance(rv, self.response_class):
            return rv
        if isinstance(rv, basestring):
            return self.response_class(rv)
        if isinstance(rv, tuple):
            return self.response_class(*rv)
        return self.response_class.force_type(rv, request.environ)
```

`Flask.make_response(rv)` 将参数 `rv` 转换为 `self.response_class` 对象。

## 3.4 请求后置处理

迭代 `self.after_request_funcs` 中保存的函数，逐个执行，返回最后执行完的数据。

添加函数的方式为 `Flask.after_request` 装饰器。

## 3.5 返回可迭代对象

Flask 默认的 `self.response_class` 类继承自 `werkzeug.Response`，这个类的 `__call__` 方法接收 `environ, start_response` 两个参数，并返回一个迭代器。

```python{.line-numbers}
class BaseResponse(object):

    def __call__(self, environ, start_response):
        app_iter, status, headers = self.get_wsgi_response(environ)
        start_response(status, headers)
        return app_iter
```

当调用 `response(environ, start_response)` 时返回一个迭代器。

# 四、Flask 如何处理并发请求？

`Flask.wsgi_app` 中的 `with self.request_context(environ)` 有什么用？

`self.request_context` 返回 `_RequestContext` 的实例化对象。

```python{.line-numbers}
class _RequestContext(object):

    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None

    def __enter__(self):
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()

```

进入 `with` 语句体时，会执行 `__enter__` 方法，离开 `with` 语句体时会执行 `__exit__` 方法。也就是说，在 Flask 开始处理请求前，会执行 `_request_ctx_stack.puth(self)`，处理完请求后会执行 `_request_ctx_stack.pop(self)`。

`_request_ctx_stack` 是一个全局变量，是 `LocalStack` 类的对象。

```python{.line-numbers}
class LocalStack(object):

    def __init__(self):
        self._local = Local()
        self._lock = allocate_lock()

    def __release_local__(self):
        self._local.__release_local__()

    def push(self, obj):
        self._lock.acquire()
        try:
            rv = getattr(self._local, 'stack', None)
            if rv is None:
                self._local.stack = rv = []
            rv.append(obj)
            return rv
        finally:
            self._lock.release()

    def pop(self):
        self._lock.acquire()
        try:
            stack = getattr(self._local, 'stack', None)
            if stack is None:
                return None
            elif len(stack) == 1:
                release_local(self._local)
                return stack[-1]
            else:
                return stack.pop()
        finally:
            self._lock.release()

    @property
    def top(self):
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```

`LocalStack` 在初始化时会创建一个线程锁（`self._lock = allocate_lock()`），`LocalStack.push(obj)` 首先请求加锁，获取到锁后将 `obj` 添加到列表 `self._local.stack`，如果 `self._local` 没有属性 `stack` 则将 `stack` 初始化为空列表，最后释放锁（`self._lock.release()`）。`self._local.stack` 是什么？

```python{.line-numbers}
class Local(object):
    __slots__ = ('__storage__', '__lock__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__lock__', allocate_lock())

    def __getattr__(self, name):
        self.__lock__.acquire()
        try:
            try:
                return self.__storage__[get_ident()][name]
            except KeyError:
                raise AttributeError(name)
        finally:
            self.__lock__.release()

    def __setattr__(self, name, value):
        self.__lock__.acquire()
        try:
            ident = get_ident()
            storage = self.__storage__
            if ident in storage:
                storage[ident][name] = value
            else:
                storage[ident] = {name: value}
        finally:
            self.__lock__.release()
```

在初始化 `self._local.stack` 属性时，`self._local.stack = rv = []` 等同于 `Local.__setattr__(self._local, 'stack', [])`，这里首先加锁，然后获取线程id（`ident = get_ident()`），将线程id作为字典 `self.__storage__` 的键，将 `{'stack': []}` 作为值，即：

```python{.line-numbers}
self.__storage__ = {
    "thread1": {
        "stack": []
    },
    "thread2": {
        "stack": []
    },
    ...
}
```

`LocalStack.push(obj)` 是将 `obj` 加入了对应线程的 `stack` 列表。`LocalStack.pop()` 是将 `obj` 从对应的线程的 `stack` 列表移除。另外还有 `LocalStack.top`，返回对应线程存在 `stack` 中的对象。

为什么要如此大费周章的做这件事呢？来看一个例子：

假设 Web 服务器单进程启动，启动2个线程，同一时刻有2个请求进来，每个请求都有对应的 `environ` 数据，如果直接赋值给同一个变量，后一个请求会覆盖前一个请求的数据，因为线程数据是共享的。要如何保存这两个请求的数据？并在需要的时候能正确取出？Flask 的做法是设置一个全局变量 `_request_ctx_stack`，存储数据最终用一个字典 `self.__storage__`，将不同线程的请求数据用线程id作为 key，请求数据存在线程id对应的 `stack` 列表中。

```python{.line-numbers}
self.__storage__ = {
    "thread1": {
        "stack": [obj1]
    },
    "thread2": {
        "stack": [obj2]
    },
    ...
}
```

Flask 提供了非常便捷的方式来获取当前请求中的 Flask 对象、请求、session、全局变量等数据：`current_app, request, session, g`。


```python{.line-numbers}
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

其本质是返回存在于 `_request_ctx_stack.top` 中对应的属性，返回的数据是与当前线程id相对应的，实现了线程数据隔离。

`LocalProxy` 是封装 `local` 的代理对象，能够保护 `local` 对象，防止其被意外修改，对应设计模式中的代理模式。

# 五、总结

回过头看看 Flask 的常见用法：

```python{.line-numbers}
from flask import Flask, request, session


app = Flask(__name__)


@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != USERNAME:
            error = 'Invalid username'
        elif request.form['password'] != PASSWORD:
            error = 'Invalid password'
        else:
            session['logged_in'] = True
            flash('You were logged in')
            return redirect(url_for('show_entries'))
    return render_template('login.html', error=error)


if __name__ == '__main__':
    app.run()
```

首先实例化一个 Flask 对象 `app`，使用装饰器 `route` 添加 URL 与处理函数，执行 `app.run()`，启动服务。

访问 `localhost:5000/login`，服务接收到请求后会将 `request, current_app` 等数据保存至线程id对应的 `stack` 中，然后进行前置处理（`self.preprocess_request`），解析 URL，获取 endpoint 与参数，通过 endpoint 从 `self.view_functions` 中获取对应的处理函数，处理完后将返回值转换为响应对象，进行后置处理（`self.process_response`），最后返回可迭代对象（`response(environ, start_response)`），之后便是服务器处理的部分了。

> 版本：`python 2.7, werkzeug==0.6.1, Flask==0.1`
> 
> 参考文献：
> 
> [1] Eby, P.J. (2010). PEP 3333 – Python Web Server Gateway Interface v1.0.1. Retrieved February 18, 2023, from https://peps.python.org/pep-3333/.
> [2] What is an 'endpoint' in Flask?.(n.d). Retrieved February 18, 2023, from https://stackoverflow.com/questions/19261833/what-is-an-endpoint-in-flask/19262349#19262349.
>
>
> 本文源码：
