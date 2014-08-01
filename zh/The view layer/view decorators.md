视图装饰器
==============

Django提供了几个装饰器，他们应用于视图来支持各种HTTP的特征。

## 允许的HTTP方法

`django.views.decorators.http`中的装饰器用于基于请求方法限制访问视图。如果条件不符合这些装饰器将会返回一个`django.http.HttpResponseNotAllowed`。

**require_http_methods(request_method_list)**

装饰器要求一个视图进接受特定的请求方法。用法：
```python

from django.views.decorators.http import require_http_methods

@require_http_methods(["GET","POST"])
def my_view(request):
	# I can assume now that only GET or POST requests make it this far
	# ...
	pass
```

需要注意的是请求方法应该是大写的。

**require_GET()**

装饰器要求视图仅接受GET方法。

**require_POST()**

装饰器要求视图仅接受POST方法。

**require_safe()**

装饰器要求视图仅接受GET和HEAD方法。这些方法通常认为是“安全的”，因为它们没有采取动作的意义而是接收请求的资源。

**注意**

对于HEAD请求，Django将自动截去响应的内容，而保留头部不变，因此你完全可以在你的视图中像处理GET请求一样处理HEAD请求。因此一些软件，例如link checkers,依靠HEAD请求，你完全可以使用require_safe来代替require_GET。

## 条件视图处理

`django.views.decorators.http`中的下列装饰器可以用于在特别的视图上控制缓存的行为。

**condition(etag_func=None, last_modified_func=None)**

**etag(etag_func)**

**last_modified(last_modified_func)**

这些装饰器可以用于生成ETag和Last-Modified头；查看 [conditional view processing](https://docs.djangoproject.com/en/1.6/topics/conditional-view-processing/)了解更多。

## GZip 压缩

`django.views.decorators.gzip`中的装饰器控制在每个视图的基础上内容的压缩。

**gzip_page()**

如果浏览器允许gzip压缩，这个装饰器压缩content。它相应地会设置vary头部，这样，缓存会根据他们的Accept-Encoding头储存。

## Vary 头部

`django.views.decorators.vary` 中的装饰器用于控制基于特定请求头的缓存。

**vary_on_cookie(func)**

**vary_on_headers(\*headers)**

这个vary头部定义了当建立cache key时，应该考虑哪一种请求头的缓存机制。

查看 [using vary headers](https://docs.djangoproject.com/en/1.6/topics/cache/#using-vary-headers)。