---
layout: post
title: "write a simple webframework with python"
date: 2016-06-30 08:40:15 +0800
comments: true
categories: python
tags: [python]
---
#### A little words before

After read the pep333 for wsgi interface,I wrote a simple webframework.It should be several versions of this, but I can not upload the files from the company, so this is the final version I wrote and no other simpler versions.

<!--more-->

#### Structure of the framework

There are four files now:

- runserver.py: Give me a container that I can run the application.
- snow.py: No any meanings of this name, wrote it which flash in my head.Don't be serious.This files is a main files which contains the URL routes and dispatch the request to the right function.
- urls.py: URL route for snow.py
- views.py: Function that process the request.

According to the pep333, the application object is simply a callable object that accepts two arguments.The term "Object" may be a function,method,class or instance with a __call__ method. Here I used a class, and return a iterable object.

#### Source code

Here is my code:

runserver.py
```python
from wsgiref.simple_server import make_server
from snow import snow
from urls import urls
from views import views

app = snow(urls,views)

if __name__ == '__main__':
        httpd = make_server('',8000,app)
        sa = httpd.socket.getsockname()
        print 'http://{0}:{1}/'.format(*sa)

        httpd.serve_forever()
```
snow.py
```python
"""
snow.py
Simple python web framework wrote by myself for learn
"""
import re
class snow(object):
        headers = []

        def __init__(self,urls=(),views=()):
                self.urls = urls
                self.views = views
                self.status = '200 OK'

        def __call__(self,environ,start_response):
                del self.headers[:]
                result = self.delegate(environ)
                start_response(self.status,self.headers)
                if isinstance(result,basestring):
                        return iter([result])
                else:
                        return iter(result)

        def delegate(self,environ):
                path = environ['PATH_INFO']
                method = environ['REQUEST_METHOD']

                for pattern,name in self.urls:
                        m = re.match('^' + pattern + '$', path)
                        if m:
                                args = m.groups()
                                if hasattr(self.views,name):
                                        func = getattr(self.views,name)
                                        return func(self.views(),*args)

                return self.notFound()

        def notFound(self):
                self.status = '404 Not Found'
                self.headers.append(('Content-type','text/plain'))
                return "Not Found\n"
```
views.py and urls.py
```python
class views(object):

    def index(self):
        return "Welcome!\n"

    def hello(self,name):
        return "Hello 1111%s \n" % name
        
urls = (
        ("/","index"),
        ("/hello/(.*)","hello"),
)
```
#### Result you can get

You can execute the python runserver.py to start the server, and type the http://127.0.0.1:8000 on you browser to get "Welcome" response.

It is a simple framework, no logs,no error handling etc,I'll Try to Reinvent the Wheel if I have enough time.



