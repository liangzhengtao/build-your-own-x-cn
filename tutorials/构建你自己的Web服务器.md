# 构建你自己的 Web 服务器

> 从零开始，用 Python 构建一个支持 HTTP/1.1 的 Web 服务器。

## 📋 项目简介

Web 服务器是互联网的基础设施。每次你在浏览器中打开一个网页，背后都有一个 Web 服务器在处理请求。通过从零构建一个 Web 服务器，你将深入理解 HTTP 协议、TCP/IP 网络编程、并发处理等核心概念。

本教程将引导你构建 `TinyServer`，支持：
- HTTP/1.0 和 HTTP/1.1 协议
- GET / POST / PUT / DELETE 请求方法
- 静态文件服务
- 路由系统
- 简单的模板渲染
- 多线程并发处理
- MIME 类型识别

## 🛠️ 技术栈选择

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Python 3.10+ | 标准库即可，无需第三方依赖 |
| 网络 | socket 模块 | 底层 TCP 通信 |
| 并发 | threading / asyncio | 多线程或异步 I/O |
| 协议 | HTTP/1.1 (RFC 7230-7235) | 手动解析 |
| 测试 | curl / 浏览器 | 手动测试 |

## 📖 核心概念解释

### 1. HTTP 协议基础

HTTP 是一种基于 TCP 的请求-响应协议：

```
客户端（浏览器）                     服务器
     │                                │
     │──── TCP 三次握手 ────────────→ │
     │                                │
     │──── HTTP 请求 ───────────────→ │
     │     GET /index.html HTTP/1.1   │
     │     Host: example.com          │
     │                                │
     │←──── HTTP 响应 ───────────────│
     │     HTTP/1.1 200 OK            │
     │     Content-Type: text/html    │
     │     <html>...</html>           │
     │                                │
     │──── TCP 四次挥手 ────────────→ │
```

### 2. HTTP 请求格式

```
┌─────────────────────────────────────────┐
│ 请求行                                   │
│ 方法 [空格] URL [空格] 版本 \r\n          │
├─────────────────────────────────────────┤
│ 请求头                                   │
│ Header-Name: Header-Value \r\n          │
│ ...更多头... \r\n                        │
├─────────────────────────────────────────┤
│ \r\n（空行，分隔头部和正文）               │
├─────────────────────────────────────────┤
│ 请求体（POST/PUT 时存在）                 │
└─────────────────────────────────────────┘
```

### 3. HTTP 响应格式

```
┌─────────────────────────────────────────┐
│ 状态行                                   │
│ 版本 [空格] 状态码 [空格] 原因短语 \r\n    │
├─────────────────────────────────────────┤
│ 响应头                                   │
│ Content-Type: text/html \r\n            │
│ Content-Length: 1234 \r\n               │
├─────────────────────────────────────────┤
│ \r\n（空行）                              │
├─────────────────────────────────────────┤
│ 响应体                                   │
└─────────────────────────────────────────┘
```

### 4. 常见状态码

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | OK | 请求成功 |
| 301 | Moved Permanently | 永久重定向 |
| 304 | Not Modified | 资源未修改 |
| 400 | Bad Request | 请求格式错误 |
| 404 | Not Found | 资源不存在 |
| 405 | Method Not Allowed | 方法不允许 |
| 500 | Internal Server Error | 服务器内部错误 |

### 5. 并发模型

```
模型 1: 单线程（阻塞）
  请求1 ──处理──→ 请求2 ──处理──→ 请求3

模型 2: 多线程（每请求一线程）
  请求1 ──处理──┐
  请求2 ──处理──┤→ 汇合
  请求3 ──处理──┘

模型 3: 异步 I/O（事件循环）
  请求1 ──→ 等待I/O ──→ 请求2 ──→ 等待I/O ──→ 完成1 ──→ 完成2
```

## 🚀 分步实现指南

### 第 1 步：最简单的 HTTP 服务器

```python
# server_basic.py - 最简 HTTP 服务器
import socket

def simple_server():
    """最简单的 HTTP 服务器 - 处理单个请求"""
    # 创建 TCP socket
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    # 绑定地址和端口
    server_socket.bind(('127.0.0.1', 8080))
    server_socket.listen(1)

    print("服务器启动在 http://127.0.0.1:8080")

    while True:
        # 接受客户端连接
        client_socket, address = server_socket.accept()
        print(f"连接来自: {address}")

        # 接收请求数据
        request_data = client_socket.recv(4096).decode('utf-8')
        print(f"请求:\n{request_data}")

        # 构造响应
        response_body = "<html><body><h1>你好，世界！</h1></body></html>"
        response = (
            "HTTP/1.1 200 OK\r\n"
            "Content-Type: text/html; charset=utf-8\r\n"
            f"Content-Length: {len(response_body.encode('utf-8'))}\r\n"
            "Connection: close\r\n"
            "\r\n"
            f"{response_body}"
        )

        # 发送响应
        client_socket.sendall(response.encode('utf-8'))
        client_socket.close()

if __name__ == "__main__":
    simple_server()
```

### 第 2 步：HTTP 请求解析器

```python
# http/request.py
from dataclasses import dataclass, field
from typing import Dict, Optional
from urllib.parse import urlparse, parse_qs

@dataclass
class HTTPRequest:
    """HTTP 请求对象"""
    method: str = "GET"
    path: str = "/"
    version: str = "HTTP/1.1"
    headers: Dict[str, str] = field(default_factory=dict)
    body: Optional[str] = None
    query_params: Dict[str, list] = field(default_factory=dict)
    client_address: tuple = ('', 0)

    @property
    def host(self) -> str:
        return self.headers.get('Host', 'localhost')

    @property
    def content_length(self) -> int:
        return int(self.headers.get('Content-Length', 0))

    @property
    def content_type(self) -> str:
        return self.headers.get('Content-Type', '')


class HTTPRequestParser:
    """HTTP 请求解析器"""

    @staticmethod
    def parse(raw_data: bytes, client_address: tuple = ('', 0)) -> HTTPRequest:
        """解析原始字节数据为 HTTPRequest 对象"""
        text = raw_data.decode('utf-8', errors='replace')

        # 分离头部和正文
        if '\r\n\r\n' in text:
            header_part, body_part = text.split('\r\n\r\n', 1)
        else:
            header_part = text
            body_part = ''

        # 解析请求行
        lines = header_part.split('\r\n')
        if not lines:
            raise ValueError("空请求")

        request_line = lines[0].split(' ', 2)
        if len(request_line) < 2:
            raise ValueError(f"无效的请求行: {request_line}")

        method = request_line[0].upper()
        path = request_line[1]
        version = request_line[2] if len(request_line) > 2 else 'HTTP/1.1'

        # 解析 URL 和查询参数
        parsed_url = urlparse(path)
        path = parsed_url.path
        query_params = parse_qs(parsed_url.query)

        # 解析请求头
        headers = {}
        for line in lines[1:]:
            if ':' in line:
                key, value = line.split(':', 1)
                headers[key.strip()] = value.strip()

        return HTTPRequest(
            method=method,
            path=path,
            version=version,
            headers=headers,
            body=body_part if body_part else None,
            query_params=query_params,
            client_address=client_address,
        )
```

### 第 3 步：HTTP 响应构建器

```python
# http/response.py
from dataclasses import dataclass
from typing import Dict, Optional
import time

# 状态码到原因短语的映射
STATUS_PHRASES = {
    200: "OK",
    201: "Created",
    301: "Moved Permanently",
    304: "Not Modified",
    400: "Bad Request",
    403: "Forbidden",
    404: "Not Found",
    405: "Method Not Allowed",
    500: "Internal Server Error",
}

# 常见 MIME 类型
MIME_TYPES = {
    '.html': 'text/html; charset=utf-8',
    '.css': 'text/css; charset=utf-8',
    '.js': 'application/javascript; charset=utf-8',
    '.json': 'application/json; charset=utf-8',
    '.xml': 'application/xml',
    '.txt': 'text/plain; charset=utf-8',
    '.png': 'image/png',
    '.jpg': 'image/jpeg',
    '.jpeg': 'image/jpeg',
    '.gif': 'image/gif',
    '.svg': 'image/svg+xml',
    '.ico': 'image/x-icon',
    '.pdf': 'application/pdf',
    '.zip': 'application/zip',
    '.woff': 'font/woff',
    '.woff2': 'font/woff2',
}

@dataclass
class HTTPResponse:
    """HTTP 响应对象"""
    status_code: int = 200
    headers: Dict[str, str] = None
    body: Optional[bytes] = None
    version: str = "HTTP/1.1"

    def __post_init__(self):
        if self.headers is None:
            self.headers = {}

    def set_header(self, key: str, value: str):
        self.headers[key] = value
        return self

    def set_body(self, body: str | bytes, content_type: str = None):
        if isinstance(body, str):
            self.body = body.encode('utf-8')
        else:
            self.body = body

        if content_type:
            self.headers['Content-Type'] = content_type

        self.headers['Content-Length'] = str(len(self.body))
        return self

    def build(self) -> bytes:
        """构建完整的 HTTP 响应字节流"""
        phrase = STATUS_PHRASES.get(self.status_code, "Unknown")

        # 状态行
        status_line = f"{self.version} {self.status_code} {phrase}\r\n"

        # 默认头部
        self.headers.setdefault('Date', time.strftime('%a, %d %b %Y %H:%M:%S GMT', time.gmtime()))
        self.headers.setdefault('Server', 'TinyServer/1.0')

        # 构建头部
        header_lines = ''.join(
            f"{key}: {value}\r\n" for key, value in self.headers.items()
        )

        # 组装响应
        response = (status_line + header_lines + "\r\n").encode('utf-8')
        if self.body:
            response += self.body

        return response

    # ---- 便捷工厂方法 ----

    @classmethod
    def ok(cls, body: str = "", content_type: str = "text/html; charset=utf-8"):
        resp = cls(200)
        resp.set_body(body, content_type)
        return resp

    @classmethod
    def not_found(cls, message: str = "<h1>404 - 页面未找到</h1>"):
        resp = cls(404)
        resp.set_body(message, "text/html; charset=utf-8")
        return resp

    @classmethod
    def error(cls, status_code: int, message: str = ""):
        resp = cls(status_code)
        phrase = STATUS_PHRASES.get(status_code, "Error")
        body = f"<h1>{status_code} - {phrase}</h1><p>{message}</p>"
        resp.set_body(body, "text/html; charset=utf-8")
        return resp
```

### 第 4 步：路由系统

```python
# routing/router.py
import re
from typing import Callable, List, Tuple, Optional
from http.request import HTTPRequest
from http.response import HTTPResponse

class Route:
    """路由条目"""

    def __init__(self, method: str, pattern: str, handler: Callable):
        self.method = method
        self.pattern = pattern
        self.handler = handler
        self.param_names = []
        self.regex = self._compile_pattern(pattern)

    def _compile_pattern(self, pattern: str) -> re.Pattern:
        """将路由模式编译为正则表达式
        /users/:id/posts/:pid → /users/(?P<id>[^/]+)/posts/(?P<pid>[^/]+)
        """
        param_pattern = r':(\w+)'
        self.param_names = re.findall(param_pattern, pattern)
        regex_str = re.sub(param_pattern, r'(?P<\1>[^/]+)', pattern)
        regex_str = f"^{regex_str}$"
        return re.compile(regex_str)

    def match(self, method: str, path: str) -> Optional[dict]:
        """匹配请求，返回路径参数字典或 None"""
        if self.method != method:
            return None
        m = self.regex.match(path)
        if m:
            return m.groupdict()
        return None


class Router:
    """URL 路由器"""

    def __init__(self):
        self.routes: List[Route] = []
        self.before_middlewares: List[Callable] = []
        self.after_middlewares: List[Callable] = []

    def add_route(self, method: str, path: str, handler: Callable):
        """注册路由"""
        self.routes.append(Route(method.upper(), path, handler))

    def get(self, path: str):
        """装饰器：注册 GET 路由"""
        def decorator(handler: Callable):
            self.add_route('GET', path, handler)
            return handler
        return decorator

    def post(self, path: str):
        """装饰器：注册 POST 路由"""
        def decorator(handler: Callable):
            self.add_route('POST', path, handler)
            return handler
        return decorator

    def put(self, path: str):
        """装饰器：注册 PUT 路由"""
        def decorator(handler: Callable):
            self.add_route('PUT', path, handler)
            return handler
        return decorator

    def delete(self, path: str):
        """装饰器：注册 DELETE 路由"""
        def decorator(handler: Callable):
            self.add_route('DELETE', path, handler)
            return handler
        return decorator

    def before_request(self, handler: Callable):
        """注册请求前中间件"""
        self.before_middlewares.append(handler)
        return handler

    def after_request(self, handler: Callable):
        """注册请求后中间件"""
        self.after_middlewares.append(handler)
        return handler

    def resolve(self, request: HTTPRequest) -> Tuple[Optional[Callable], dict]:
        """解析请求，返回处理函数和路径参数"""
        for route in self.routes:
            params = route.match(request.method, request.path)
            if params is not None:
                return route.handler, params

        return None, {}
```

### 第 5 步：主服务器

```python
# server.py
import socket
import threading
import os
import mimetypes
from http.request import HTTPRequestParser
from http.response import HTTPResponse, MIME_TYPES
from routing.router import Router

class TinyServer:
    """多线程 HTTP Web 服务器"""

    def __init__(self, host: str = '127.0.0.1', port: int = 8080,
                 static_dir: str = 'static', max_threads: int = 64):
        self.host = host
        self.port = port
        self.static_dir = os.path.abspath(static_dir)
        self.router = Router()
        self.max_threads = max_threads
        self.running = False
        self._thread_pool = threading.BoundedSemaphore(max_threads)

    def route(self, path: str, methods: list = None):
        """路由装饰器"""
        if methods is None:
            methods = ['GET']

        def decorator(handler):
            for method in methods:
                self.router.add_route(method, path, handler)
            return handler
        return decorator

    def start(self):
        """启动服务器"""
        self.running = True

        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind((self.host, self.port))
        server_socket.listen(128)
        server_socket.settimeout(1.0)

        print(f"╔══════════════════════════════════════╗")
        print(f"║  TinyServer v1.0                     ║")
        print(f"║  http://{self.host}:{self.port}            ║")
        print(f"║  静态文件: {self.static_dir:<25s}║")
        print(f"║  最大并发: {self.max_threads:<25d}║")
        print(f"╚══════════════════════════════════════╝")
        print(f"按 Ctrl+C 停止服务器\n")

        try:
            while self.running:
                try:
                    client_socket, address = server_socket.accept()
                    self._thread_pool.acquire()
                    thread = threading.Thread(
                        target=self._handle_client,
                        args=(client_socket, address),
                        daemon=True
                    )
                    thread.start()
                except socket.timeout:
                    continue
        except KeyboardInterrupt:
            print("\n正在关闭服务器...")
        finally:
            self.running = False
            server_socket.close()

    def _handle_client(self, client_socket: socket.socket, address: tuple):
        """处理单个客户端连接"""
        try:
            # 接收数据
            request_data = b''
            while True:
                chunk = client_socket.recv(4096)
                if not chunk:
                    break
                request_data += chunk

                # 检查是否收到完整请求
                if b'\r\n\r\n' in request_data:
                    header_end = request_data.index(b'\r\n\r\n') + 4
                    headers = request_data[:header_end].decode('utf-8', errors='replace')

                    # 获取 Content-Length
                    content_length = 0
                    for line in headers.split('\r\n'):
                        if line.lower().startswith('content-length:'):
                            content_length = int(line.split(':')[1].strip())
                            break

                    if len(request_data) >= header_end + content_length:
                        break

            if not request_data:
                client_socket.close()
                return

            # 解析请求
            request = HTTPRequestParser.parse(request_data, address)
            print(f"[{address[0]}] {request.method} {request.path}")

            # 处理请求
            response = self._process_request(request)

            # 发送响应
            client_socket.sendall(response.build())

        except Exception as e:
            print(f"错误处理 {address}: {e}")
            error_response = HTTPResponse.error(500, str(e))
            try:
                client_socket.sendall(error_response.build())
            except:
                pass
        finally:
            self._thread_pool.release()
            client_socket.close()

    def _process_request(self, request) -> HTTPResponse:
        """处理 HTTP 请求"""
        # 执行前置中间件
        for middleware in self.router.before_middlewares:
            result = middleware(request)
            if isinstance(result, HTTPResponse):
                return result

        # 查找路由处理函数
        handler, params = self.router.resolve(request)

        if handler:
            # 调用路由处理函数
            try:
                response = handler(request, **params)
                if isinstance(response, str):
                    response = HTTPResponse.ok(response)
                elif isinstance(response, dict):
                    import json
                    body = json.dumps(response, ensure_ascii=False)
                    response = HTTPResponse.ok(body, "application/json; charset=utf-8")
                return response
            except Exception as e:
                return HTTPResponse.error(500, str(e))
        else:
            # 尝试静态文件服务
            return self._serve_static(request.path)

    def _serve_static(self, path: str) -> HTTPResponse:
        """提供静态文件服务"""
        if path == '/':
            path = '/index.html'

        # 安全检查：防止目录遍历
        file_path = os.path.normpath(os.path.join(self.static_dir, path.lstrip('/')))
        if not file_path.startswith(self.static_dir):
            return HTTPResponse.error(403, "禁止访问")

        if os.path.isfile(file_path):
            ext = os.path.splitext(file_path)[1].lower()
            content_type = MIME_TYPES.get(ext, 'application/octet-stream')

            with open(file_path, 'rb') as f:
                content = f.read()

            response = HTTPResponse(200)
            response.set_header('Content-Type', content_type)
            response.body = content
            response.headers['Content-Length'] = str(len(content))
            return response

        return HTTPResponse.not_found()
```

### 第 6 步：完整应用示例

```python
# app.py - 使用 TinyServer 构建 Web 应用
from server import TinyServer
from http.response import HTTPResponse
from http.request import HTTPRequest
import json
import time

# 创建服务器实例
app = TinyServer(port=8080, static_dir='./static')

# ---- 中间件 ----

@app.router.before_request
def log_middleware(request: HTTPRequest):
    """请求日志中间件"""
    request.start_time = time.time()
    print(f"→ {request.method} {request.path}")
    return None  # 继续处理

@app.router.after_request
def cors_middleware(request, response):
    """CORS 中间件"""
    response.set_header('Access-Control-Allow-Origin', '*')
    return response

# ---- 路由处理 ----

@app.route('/', methods=['GET'])
def index(request):
    return """
    <!DOCTYPE html>
    <html>
    <head><title>TinyServer</title></head>
    <body>
        <h1>欢迎来到 TinyServer!</h1>
        <p>一个从零构建的 Web 服务器</p>
        <ul>
            <li><a href="/api/hello">API 测试</a></li>
            <li><a href="/api/time">服务器时间</a></li>
            <li><a href="/echo?msg=你好">Echo 测试</a></li>
        </ul>
    </body>
    </html>
    """

@app.route('/api/hello', methods=['GET'])
def hello(request):
    name = request.query_params.get('name', ['World'])[0]
    return {"message": f"Hello, {name}!", "status": "ok"}

@app.route('/api/time', methods=['GET'])
def server_time(request):
    import datetime
    return {
        "time": datetime.datetime.now().isoformat(),
        "timezone": time.tzname[0]
    }

@app.route('/api/echo', methods=['GET', 'POST'])
def echo(request):
    if request.method == 'GET':
        return {"method": "GET", "params": request.query_params}
    else:
        return {"method": "POST", "body": request.body}

@app.route('/api/users/:id', methods=['GET'])
def get_user(request, id):
    # 模拟用户数据
    users = {
        "1": {"id": 1, "name": "张三", "email": "zhangsan@example.com"},
        "2": {"id": 2, "name": "李四", "email": "lisi@example.com"},
    }
    user = users.get(id)
    if user:
        return user
    return HTTPResponse.not_found(json.dumps({"error": "用户不存在"}))

@app.route('/api/users', methods=['POST'])
def create_user(request):
    if request.body:
        data = json.loads(request.body)
        return HTTPResponse(201).set_body(
            json.dumps({"id": 3, **data}, ensure_ascii=False),
            "application/json; charset=utf-8"
        )
    return HTTPResponse.error(400, "请求体不能为空")

# ---- 启动 ----

if __name__ == "__main__":
    app.start()
```

### 第 7 步：使用 asyncio 实现高并发版

```python
# server_async.py - 异步 HTTP 服务器
import asyncio
from http.request import HTTPRequestParser
from http.response import HTTPResponse

class AsyncTinyServer:
    """基于 asyncio 的高性能 HTTP 服务器"""

    def __init__(self, host='127.0.0.1', port=8080):
        self.host = host
        self.port = port
        self.routes = {}

    async def handle_client(self, reader: asyncio.StreamReader,
                            writer: asyncio.StreamWriter):
        """处理单个客户端连接"""
        try:
            # 读取请求数据
            request_data = b''
            while True:
                chunk = await reader.read(4096)
                if not chunk:
                    break
                request_data += chunk
                if b'\r\n\r\n' in request_data:
                    break

            if not request_data:
                writer.close()
                return

            # 解析请求
            addr = writer.get_extra_info('peername')
            request = HTTPRequestParser.parse(request_data, addr)

            # 查找路由
            key = f"{request.method}:{request.path}"
            handler = self.routes.get(key)

            if handler:
                response = await handler(request) if asyncio.iscoroutinefunction(handler) else handler(request)
            else:
                response = HTTPResponse.not_found()

            # 发送响应
            if isinstance(response, str):
                response = HTTPResponse.ok(response)
            writer.write(response.build())
            await writer.drain()

        except Exception as e:
            error = HTTPResponse.error(500, str(e))
            writer.write(error.build())
            await writer.drain()
        finally:
            writer.close()

    def route(self, path: str, methods=None):
        if methods is None:
            methods = ['GET']
        def decorator(handler):
            for method in methods:
                self.routes[f"{method}:{path}"] = handler
            return handler
        return decorator

    async def start(self):
        server = await asyncio.start_server(
            self.handle_client, self.host, self.port
        )
        print(f"异步服务器启动在 http://{self.host}:{self.port}")
        async with server:
            await server.serve_forever()

# 使用示例
async_app = AsyncTinyServer()

@async_app.route('/')
async def index(request):
    await asyncio.sleep(0)  # 模拟异步操作
    return "<h1>异步 TinyServer</h1>"

if __name__ == "__main__":
    asyncio.run(async_app.start())
```

## 🎯 进阶挑战

### 初级挑战
- [ ] 实现 Cookie 的读写
- [ ] 添加会话（Session）支持
- [ ] 支持文件上传（multipart/form-data）

### 中级挑战
- [ ] 实现 HTTPS（TLS/SSL）
- [ ] 添加 WebSocket 支持
- [ ] 实现连接池和 Keep-Alive
- [ ] 支持 chunked 传输编码

### 高级挑战
- [ ] 实现反向代理功能
- [ ] 添加请求速率限制
- [ ] 实现完整的 CGI/FastCGI 协议
- [ ] 支持 HTTP/2 协议

## 📚 参考资源

- [RFC 7230-7235](https://httpwg.org/specs/) — HTTP/1.1 协议规范
- [Beej's Network Programming Guide](https://beej.us/guide/bgnet/) — 网络编程入门经典
- [《HTTP 权威指南》](https://book.douban.com/subject/10746243/) — HTTP 协议深入理解
- [Tornado 源码](https://github.com/tornadoweb/tornado) — Python 异步 Web 框架
- [Python socket 编程文档](https://docs.python.org/3/library/socket.html)

---

**难度**: ⭐⭐⭐ | **预计时长**: 8+ 小时 | **语言**: Python

> 💡 **提示**：从最简单的"返回 Hello World"开始，逐步添加功能。每增加一个新特性都要测试。用 `curl -v` 可以看到完整的请求和响应。
