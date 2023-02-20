# 〇、前言

## 往期解读

- [Flask 渐进式源码解读: 0.1](https://github.com/yyywang/flask-docs/blob/main/flask%200.1%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)

## 本期导读

Flask 0.2 提供了快捷生成 JSON 响应的函数：`jsonify`，如何实现的呢？网络中的字节流数据如何传递到 Flask，Flask 又是如何生成字节流数据返回给客户端的？我们从服务器接收到 HTTP 消息说起。

# 一、服务器接收到 HTTP 消息

HTTP 消息是“一问一答”的形式，客户端发问（请求），服务端回答（响应）。先有客户端还是先有服务端？

去小卖铺买冰淇淋，如果老板不在店里，在店里喊：“老板，来个冰淇淋”，老板能有回复吗？不能，因为老板没有在接收消息，必须要老板在线，处于接收消息的状态，我们发出的“请求”，才能得到“响应”。客户端与服务端也是同样的道理，必须先在服务端监听请求，然后才能收到客户端发送的请求。

客户端发送的 HTTP 消息有目标地址：主机:端口，是基于 TCP/IP 协议传输的，socket 把 TCP/IP 层复杂的操作抽象封装为了几个简单的接口，供应用层调用实现程序在网络中的通信。

在 UNIX 操作系统中，socket 就是一个文件，在服务端调用 socket 接口监听 TCP 请求，实际上是创建了一个可读的文件，当文件中有数据被写入时，就收到了客户端发来的请求。

《[Flask 渐进式源码解读: 0.1](https://github.com/yyywang/flask-docs/blob/main/flask%200.1%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)》中说到，执行 `serve_forever` 即可启动 Flask 服务，来详细看看：

```python
def _eintr_retry(func, *args):
    while True:
        try:
            return func(*args)
        except (OSError, select.error) as e:
            if e.args[0] != errno.EINTR:
                raise


class BaseServer:

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

`r, w, e = _eintr_retry(select.select, [self], [], [], poll_interval)` 相当于执行 `select.select([self], [], [], poll_)`，这是一个系统调用，返回值是三个列表，包含已就绪对象，若返回值 `r` 非空，表示已成功创建可读的 socket 文件，即启动了 socket 服务，开始监听请求。

```python
if self in r:
    self._handle_request_noblock()
```

`if self in r` 为 `True` 表示 socket 监听服务已启动，服务启动后会不断执行 `self._handle_request_noblock()`。

```python
def _handle_request_noblock(self):
    try:
        request, client_address = self.get_request()
    except socket.error:
        return
    if self.verify_request(request, client_address):
        try:
            self.process_request(request, client_address)
        except:
            self.handle_error(request, client_address)
            self.shutdown_request(request)
    else:
        self.shutdown_request(request)


class TCPServer(BaseServer):

    def get_request(self):
        return self.socket.accept()

    def process_request(self, request, client_address):
        self.finish_request(request, client_address)
        self.shutdown_request(request)

    def finish_request(self, request, client_address):
        self.RequestHandlerClass(request, client_address, self)
```

`self._handle_request_noblock()` 中调用 `self.get_request()`，`self.get_request()` 调用 `socket.accept()`，这个函数被动接受 TCP 客户端的连接，等待连接的到来（阻塞式，如果没有连接到来，程序会停留在这个地方，直到请求到来才会往后执行）。

# 二、HTTP 消息在 Flask 中的流动

接收到请求后，调用 `self.process_request()` -> `self.finish_request()` -> `self.RequestHandlerClass(request, client_address, self)`，在 [Flask 渐进式源码解读: 0.1，二、Flask 如何收到请求？](https://github.com/yyywang/flask-docs/blob/main/flask%200.1%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md) 中分析过，调用 `self.RequestHandlerClass(request, client_address, self)`，后续会调用 `WSGIRequestHandler.__init__()` -> `BaseRequestHandler.__init__()` -> `self.handle()` -> `Flask.__call__()`，这将请求传递至 Flask 处理。

`BaseRequsetHandler.__init__()` 中调用 `self.setup()`，这个方法被 `StreamRequestHandler` 覆写：

```python

class StreamRequestHandler(BaseRequestHandler):
    rbufsize = -1
    wbufsize = 0

    timeout = None

    disable_nagle_algorithm = False

    def setup(self):
        self.connection = self.request
        if self.timeout is not None:
            self.connection.settimeout(self.timeout)
        if self.disable_nagle_algorithm:
            self.connection.setsockopt(socket.IPPROTO_TCP,
                                       socket.TCP_NODELAY, True)
        self.rfile = self.connection.makefile('rb', self.rbufsize)
        self.wfile = self.connection.makefile('wb', self.wbufsize)
```

其创建了两个 io 流：`self.rfile` 用于读，`self.wfile` 用于写。socket 接收到的请求数据，可以从 `self.rfile` 中读到，socket 要返回给客户端的响应数据，存储在 `self.wfile` 中，调用 `self.wfile.flush()` 就会向客户端发送数据。


```python
class WSGIRequestHandler(BaseHTTPRequestHandler, object):

    def run_wsgi(self):
        app = self.server.app
        environ = self.make_environ()

        def write(data):
            ...
            self.wfile.write(data)
            self.wfile.flush()

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
```

`Flask.__call__()` 返回一个可迭代对象，后续迭代，将数据写入 io 流（`write(data)`）并发送给客户端。

# 三、Flask 生成 JSON 格式响应

在响应中，`Content-Type` 标头告诉客户端实际返回的内容的内容类型。要生成 JSON 格式响应，只需要做两件事：

1. 设置标头：`Content-Type: application/json`
2. 将数据转换为 JSON 格式的字符串

```python
def jsonify(*args, **kwargs):
    return current_app.response_class(json.dumps(dict(*args, **kwargs),
        indent=None if request.is_xhr else 2), mimetype='application/json')
```

`jsonify` 函数完成了这两件事。

1. 设置标头：`mimetype='application/json'`
2. 转换格式：`json.dumps(dict(*args, **kwargs),
        indent=None if request.is_xhr else 2)`

以下例子，在路由函数中，将数据传入 `jsonify` 并返回即可生成 JSON 格式响应。

```python
@app.route('/_get_current_user')
def get_current_user():
    return jsonify(username=g.user.username,
                    email=g.user.email,
                    id=g.user.id)

"""返回的 JSON 格式响应
{
    "username": "admin",
    "email": "admin@localhost",
    "id": 42
}
"""
```

# 四、更快捷的 JSON 化数据

JSON 是前后端数据交互最流行的方式之一，如果整个 Flask 项目 API 返回数据都是 JSON 格式，每个视图函数最后都调用一次 `jsonify` 函数显的很冗余。能否不调用 `jsonify` 而直接返回 Python 原生数据类型/自定义数据类型（比如 SQLAlchemy 的 Model 类型）就能生成 JSON 化响应？来尝试对 Flask 做一些框架层面的修改。

要实现以上目标，只需把以下两个操作添加到 Flask 生成响应之前即可：

1. 设置标头：`Content-Type: application/json`
2. 将数据转换为 JSON 格式的字符串


## 4.1 设置标头：`Content-Type: application/json`

生成响应时标头 `Content-Type` 的值默认取 `Flask.response_class` 中 `default_mimetype` 属性的值，因此只需要继承响应基类并覆写 `default_mimetype = 'application/json'` ，将 Flask 响应类替换掉即可。实现如下：

```python
from werkzeug import Response as ResponseBase


class JSONResponse(ResponseBase):
    default_mimetype = 'application/json'


Flask.response_class = JSONResponse
```

## 4.2 将数据转换为 JSON 格式的字符串

视图函数返回值转换为响应体对象是在 `Flask.make_response` 方法中实现的，将 `Flask.make_response` 覆写，判断接收到的参数是否需要进行 JSON 格式化，如果需要则转换为 JSON 格式。例如自动将 `dict, list` 格式数据转化为 JSON 格式响应，实现如下：

```python
from flask import Flask as FlaskBase


class Flask(FlaskBase):

    def make_response(self, rv):
        if isinstance(rv, (dict, list)):
            rv = json.dumps(rv)
        return FlaskBase.make_response(self, rv)
```

视图函数直接返回 `dict, list` 即可生成 JSON 格式响应，不需再调用 `jsonify` 处理，示例：

```python
@app.route('/json/list')
def test_json():
    # return jsonify([1, 2, 3])
    return [1, 2, 3]


@app.route('/json/dict')
def test_json():
    # return jsonify({'hello': 'world', 'name': 'huaiyue'})
    return {'hello': 'world', 'name': 'huaiyue'}
```

以上修改实现见：https://github.com/yyywang/flask-backend-clean-architecture

> 版本：`python 2.7, werkzeug==0.6.1, Flask==0.2`
> 
> 参考文献：
> 
> [1] Flask changes. (n.d). Retrieved February 19, 2023, from https://flask.palletsprojects.com/en/2.2.x/changes/#version-0-2.
>
> [2] MDN Web Docs. (n.d). Retrieved February 20, 2023, from https://developer.mozilla.org/zh-CN/docs/Web/HTTP.
>
> 本文源码：https://github.com/yyywang/flask-docs/blob/main/Flask%200.2%20%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB:%20HTTP%20%E6%B6%88%E6%81%AF%E5%9C%A8%20Flask%20%E4%B8%AD%E7%9A%84%E6%B5%81%E5%8A%A8%E4%B8%8E%E5%A4%84%E7%90%86.md
