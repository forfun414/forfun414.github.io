---
title: lighttpd + web.py + fastcgi server
layout: post
---

# lighttpd + web.py + fastcgi(flup)服务器搭建

lighttpd作为webserver；
web.py是一个轻量级的web框架，开发者可以基于此开发出非常简单的程序，所以服务器的实现最终都是通过web.py；
fastcgi完成webserver到web.py的转换，flup其实是实现fastcgi的库。

# 参考
[wsgi tutor](http://wsgi.tutorial.codepoint.net/application-interface)
[relation bewteen server middleware and client](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface)
[install and config](http://webpy.org/cookbook/fastcgi-lighttpd)

按照安装手册，看着非常简单，但是发现web.py单独可以运行，通过fcgi却死活运行不了，于是学习了下wsgi相关原理。

# 两个严重错误及解决方法
## 1 创建socket失败
### * 现象：
lighttpd日志中有如下错误：
```
2017-02-19 16:13:56: (mod_fastcgi.c.1733) connect failed: Connection refused on unix:/tmp/fastcgi.socket-0
2017-02-19 16:13:56: (mod_fastcgi.c.2999) backend died; we'll disable it for 1 seconds and send the request to another backend instead: reconnects: 0 load: 1
2017-02-19 16:13:56: (mod_fastcgi.c.2540) unexpected end-of-file (perhaps the fastcgi process died): pid: 19465 socket: unix:/tmp/fastcgi.socket-0
2017-02-19 16:13:56: (mod_fastcgi.c.3326) response not received, request sent: 908 on socket: unix:/tmp/fastcgi.socket-0 for /index.py?, closing connection
```
### * 修改：
配置
```
"socket" => "/tmp/fastcgi.socket"
```
修改为
```
"socket" => "/var/tmp/lighttpd/fastcgi.socket",
```
创建新的目录，同时更改目录属组为lighttpd的userid(www-data)，这样就能保证lighttpd可以正常访问socket
```
mkdir -p /var/tmp/lighttpd
chown www-data:www-data /var/tmp/lighttpd
```


## 2 无法收到response
### * 现象:
lighttpd正常运行起来了，但是访问时一直无法接收到response。
### * 分析:
在主py文件中，不通过web.py，直接以fcgi的接口通过start_response方式响应，按如下方式测试
```
def wsgiapp(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Hello World!\n']

from flup.server.fcgi import WSGIServer
WSGIServer(wsgiapp).run()
```
发现能正常工作，所以问题还是出在lighttpd和web.py的交互上。
但由于通过fcgi调用，不如py调试方便，pdb也不能用，只能通过加打印来完成，最后确定安装的web.py软件有问题。
/usr/local/lib/python2.7/dist-packages/web.py-0.40_dev0-py2.7.egg/web 文件wsgi.py 34行
```
def runwsgi(func):
    if ('PHP_FCGI_CHILDREN' in os.environ #lighttpd fastcgi
      or 'SERVER_SOFTWARE') in os.environ:
```
第三行的括号")"明显用错地方了。

### * 修改：
修改软件中wsgi.py文件为如下内容就好了
```
def runwsgi(func):
    if ('PHP_FCGI_CHILDREN' in os.environ #lighttpd fastcgi
      or 'SERVER_SOFTWARE' in os.environ):
```


  所以安装web.py，推荐用源码编译的方式安装，不推荐用easy_install方式，这种方式安装的web很可能是有问题的。在多个ubuntu上验证都有问题，但是直接下载源码，这部分都是正确的。
