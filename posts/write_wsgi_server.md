---
layout: post
title: "手写一个满足WSGI协议的Server"
date: 2018-01-25 22:20:15 +0800
comments: true
categories: python
tags: [server,wsgi]
---

在做Web开发时，一个很重要的概念就是服务端和应用程序之间的沟通协议，比如java中的servlet，由于servlet的存在，使得用java开发的web程序既可以跑在tomcat上，也可以是jetty或是jboss。反之亦然。而在python中，对应的协议也就是WSGI协议,本文的目标就是实现一个可以支持python主流框架的web服务器，也帮助自己加强对WSGI协议的理解。
<!--more-->
实验环境：
- python3.5

#### 一个简单的服务器实现

这一节并不会直接给出一个遵循WSGI协议规范的服务器，只是单纯从如何与客户端通信的角度来考虑实现。我们都知道，HTTP协议是建立在TCP协议的基础上，所以首先我们借助python标准库中的socket来实现TCP通信。下面是我的实现代码和解释：

```python
# wsgi_a.py

import socket

HOST, PORT = '', 8888

listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
listen_socket.bind((HOST, PORT))
listen_socket.listen(1)
print('Serving HTTP on port %s ...', PORT)

# run or not
flag = True

while flag:
    try:
        client_connection, client_address = listen_socket.accept()
        request = client_connection.recv(1024)
        print(request)

        http_response = """\
HTTP/1.1 200 OK

Hello, World!
"""
        client_connection.sendall(http_response.encode())
        client_connection.close()
    except KeyboardInterrupt:
        flag = False
        print('exit')

```

这里需要说明的是关于socket的标准库中的基本函数及常量：
- socket.socket(socket.AF_INET, socket.SOCK_STREAM) 返回一个socket对象，其中第一个参数需指明IP地址类型（IPv4, IPv6, ...），第二个参数用来指明通信的协议，这里两个参数的意思分别为（IPv4, TCP）。
- socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)设置套接字重用。
- socket.bind((HOST,PORT))绑定端口
- socket.listen(1) 设置客户端的连接个数
- socket.accept() 阻塞监听

打开终端。命令行中运行该脚本并在浏览器中输入
> http://127.0.0.1:8888

即可查看结果。

#### 满足WSGI协议的服务器

简单版本的服务器仅仅只是实现了一问一答，同时将请求处理也放在了服务器里，并没有将两者分开。也没有对现有的主流框架进行支持。因此为了实现一个通用的Web服务器，根据WSGI协议，我们需要添加两个关键的部分。一个传给应用端的上下文环境，并一个是需要给应用端调用的start_response函数。详情可以参照我之前的翻译[PEP333](http://10111000.com/2017/12/22/PEP333_1/)。

```python
#wsgi_b.py

import socket
import sys, io
from datetime import date

class WSGIServer(object):
    """docstring for WSGIServer"""
    def __init__(self, host, port, application):
        self.host = host
        self.port = port

        self.listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.listen_socket.bind((self.host, self.port))
        self.listen_socket.listen(1)

        self.flag = True
        self.application = application
        print('Serving HTTP on port %s ...', self.port)

    def get_environ(self, request_data):
        env = {}

        # CGI variables
        path, server, *args = request_data
        env['REQUEST_METHOD'], env['PATH_INFO'], _= path.split()
        env['SERVER_NAME'], env['SERVER_PORT'] = self.host, str(self.port)

        # WSGI variables
        env['wsgi.version']      = (1, 0)
        env['wsgi.url_scheme']   = 'http'
        env['wsgi.input']        = io.StringIO(self.request_data.decode())
        env['wsgi.errors']       = sys.stderr
        env['wsgi.multithread']  = False
        env['wsgi.multiprocess'] = False
        env['wsgi.run_once']     = False

        return env

    def make_server(self):

        while self.flag:
            try:
                self.client_connection, self.client_address = self.listen_socket.accept()
                self.request_data = self.client_connection.recv(1024)
                request_data = self.request_data.decode().splitlines()
                env = self.get_environ(request_data)
                result = self.application(env, self.start_response)
                self.make_response(result)
            except KeyboardInterrupt:
                self.flag = False
                print('exit')

    def make_response(self, result):
        try:
            status, response_headers = self.headers_set
            response = 'HTTP/1.0 {status}\r\n'.format(status=status)
            for header in response_headers:
                response += '{0}: {1}\r\n'.format(*header)
            response += '\r\n'
            for data in result:
                response += data.decode()
            self.client_connection.sendall(response.encode())
        finally:
            self.client_connection.close()

    def start_response(self, status, response_headers, exc_info=None):
        # Add necessary server headers
        server_headers = [
            ('Date', date.today().strftime('%Y-%m-%d')),
            ('Server', 'WSGIServer 0.2'),
        ]
        self.headers_set = [status, response_headers + server_headers]
```
这个版本的服务器实现了一个简单的WSGI规范，但并不是全部，不过已经可以实现与多个框架的通信。相关的解释如下：

- WSGI的初始化参数分别为主机ip，端口及需要调用的应用程序。
- get_environ函数，从request_data中获取相应的CGI变量及WSGI变量，并传给应用程序。
- make_server，阻塞监听端口。并接收客户端传来的消息交由应用程序进行处理，最后再将响应的结果打包，转成响应格式，交付给客户端。
- make_response。将应用程序处理结果及请求头打包成响应，并关闭连接。 
- start_response，这个函数交给应用程序调用，其参数分别为状态码，响应头以及错误消息处理。

#### 常见框架测试

这里是我的测试代码，通过我们自己写的服务器，可以成功的跑起weppy，flask及一个简单的满足WSGI规范的application。

```python
# test.py

from wsgi_b import WSGIServer
from flask import Flask
from weppy import App

weppy_application = App(__name__)
flask_application = Flask(__name__)

@weppy_application.route("/")
def hello():
    return "Hello World! from weppy" 

@flask_application.route("/")
def hello():
    return "Hello World! from flask"

def application(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return [b'Hello world! from simple app \n']

if __name__ == '__main__':
    server = WSGIServer('127.0.0.1',8888,weppy_application).make_server()
```
但是这个服务器仍然还有许多需要完善的地方。不过不妨碍其做为WSGI协议的学习的补充。



