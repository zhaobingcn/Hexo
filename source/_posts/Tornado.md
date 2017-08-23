---
layout: python
title: Tornado
date: 2017-08-10 17:23:00
tags:
    - Python
    - Python Web
    - WSGI
    - Tornado
---

最近需要使用python做后台研发，因此学习了下python tornado相关的知识，在这里记录下来方便以后回顾。

# Tornado简介

Tornado是使用Python编写的一个强大的、可扩展的Web服务器。它在处理严峻的网络流量时表现得足够强健，但却在创建和编写时有着足够的轻量级，并能够被用在大量的应用和工具中。

## 第一个Tornado示例

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

application = tornado.web.Application([
    (r"/", MainHandler),
])

if __name__ == "__main__":
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()

```
在浏览器输入localhost:8888即可以得到hello world的输出，这是一个最简单的发出http get请求，并且返回字符串的例子。

## 内建方法get_argument

```python
class StoryHandler(tornado.web.RequestHandler):
    def get(self):
        name = self.get_argument("name", "admin");
        self.write("you request the story " + name);

```
第一个参数为我们定义的参数形参名，第二个参数为默认参数，如果我们传递参数将使用第二个参数作为默认值。


## 参数handlers
```python
application = tornado.web.Application([
    (r"/", MainHandler),
])
```
这里的参数handle非常重要，值得我们更加深入的研究。他是元组组成的一个列表，其中第一个参数是解析url的正则表达式，第二个参数是RequestHander类。


## 字符串服务
```python
import textwrap

import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web

from tornado.options import define, options
define("port", default=8000, help="run on the given port", type=int)

class ReverseHandler(tornado.web.RequestHandler):
    def get(self, input):
        self.write(input[::-1])

class WrapHandler(tornado.web.RequestHandler):
    def post(self):
        text = self.get_argument('text')
        width = self.get_argument('width', 40)
        self.write(textwrap.fill(text, int(width)))
        
if __name__ == "__main__":
    tornado.options.parse_command_line()
    app = tornado.web.Application(
        handlers=[
            (r"/reverse/(\w+)", ReverseHandler),
            (r"/wrap", WrapHandler)
        ]
    )
    http_server = tornado.httpserver.HTTPServer(app)
    http_server.listen(options.port)
    tornado.ioloop.IOLoop.instance().start()

```
这里有两个Handler，第一个是将字符串翻转，第二个是输入参数为字符串，并且在一定的宽度范围内显示。

你可以看到这里的get方法有一个额外的参数input。这个参数将包含匹配处理函数正则表达式第一个括号里的字符串。（如果正则表达式中有一系列额外的括号，匹配的字符串将被按照在正则表达式中出现的顺序作为额外的参数传递进来。）

## HTTP方法

一个Requesthandler类里面可以有多个HTTP方法，虽然上面我们只定义了一种HTTP方法，但是实际中我们是可以定义多种HTTP方法的。包括（GET,POST,PUT,DELETE,HEAD,OPTION）等等。下面一个就是定义了GET与POST方法的RequestHandler类。
```python
class WidgetHandler(tornado.web.RequestHandler):
    def get(self, widget_id):
        widget = retrieve_from_db(widget_id)
        self.write(widget.serialize())

    def post(self, widget_id):
        widget = retrieve_from_db(widget_id)
        widget['foo'] = self.get_argument('foo')
        save_to_db(widget)

```

# 模板和表单

模仿一个表单的提交请求，并且获取得到的参数值
```python
import os.path

import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web

from tornado.options import define, options
define("port", default=8000, help="run on the given port", type=int)

class IndexHandler(tornado.web.RequestHandler):
    def get(self):
        self.render('index.html')

class PoemPageHandler(tornado.web.RequestHandler):
    def post(self):
        noun1 = self.get_argument('noun1')
        noun2 = self.get_argument('noun2')
        verb = self.get_argument('verb')
        noun3 = self.get_argument('noun3')
        self.render('poem.html', roads=noun1, wood=noun2, made=verb,
                difference=noun3)

if __name__ == '__main__':
    tornado.options.parse_command_line()
    app = tornado.web.Application(
        handlers=[(r'/', IndexHandler), (r'/poem', PoemPageHandler)],
        template_path=os.path.join(os.path.dirname(__file__), "../templates")
    )
    http_server = tornado.httpserver.HTTPServer(app)
    http_server.listen(options.port)
    tornado.ioloop.IOLoop.instance().start()

```
index.html页面，包括form表单，用的是post请求。
```html
<!DOCTYPE html>
<html>
    <head><title>Poem Maker Pro</title></head>
    <body>
        <h1>Enter terms below.</h1>
        <form method="post" action="/poem">
        <p>Plural noun<br><input type="text" name="noun1"></p>
        <p>Singular noun<br><input type="text" name="noun2"></p>
        <p>Verb (past tense)<br><input type="text" name="verb"></p>
        <p>Noun<br><input type="text" name="noun3"></p>
        <input type="submit">
        </form>
    </body>
</html>
```

poem页面，即模板页面，使用得到的数据填充，填充一个参数的标识符是{}
```html
<!DOCTYPE html>
<html>
    <head><title>Poem Maker Pro</title></head>
    <body>
        <h1>Your poem</h1>
        <p>Two {{roads}} diverged in a {{wood}}, and I—<br>
I took the one less travelled by,<br>
And that has {{made}} all the {{difference}}.</p>
    </body>
</html>
```

self.render会开始渲染模板，但是这个模板中没有任何我们传入的参数，下面的poem页面即有我们传入参数的模板渲染，这个叫**填充**。



