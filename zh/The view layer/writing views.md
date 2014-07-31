编写视图
===========

一个视图函数，或者简短的视图函数，是一个简单的Python函数，它是根据一个Web请求返回一个Web响应。这个响应可以是Web页面的HTML文本，或者一个重定向，或者一个404错误，或者一个XML文档，或者一张图片...或者任何东西。视图自己包含任何任意的逻辑是必要的，以返回响应。这个代码可以放在你想要的任何地方，只要它在你的Python路径上。没有其他要求 - 没有"magic",可以这么说。为了放置代码到某些地方，惯例是把视图放在一个叫views.py的文件中，该文件位于你的工程或者应用目录中。

## 一个简单的视图

这里有一个视图以一个HTML文档返回当前日期和时间：

```python

from django.http import HttpResponse
import datetime

def current_datetime(request):
	now = datetime.datetime.now()
	html = "<html><bode>It is now %s.</body></html>" % now
	return HttpResponse(html)
```
让我们来逐行分析这段代码：

- 首先，我们从django.http模块中导入HttpResponse类，随后是Python的datetime库。

- 下一步，我们定义一个函数叫做current_datetime。这是一个视图函数。每个视图函数都有一个HttpRequest对象作为它的第一个参数，它通常被命名为request。
  需要注意的是，视图函数的名称并不重要。不必为了使Django识别它而以某种方式来命名它。我们在这里叫它为current_datetime，因为这个名称能清晰得指明它是干什么的。
- 视图返回一个HttpResponse对象，它包含了生成的响应。每个视图函数负责返回一个HttpResponse对象(也有例外，但我们以后会提到它们。)

**Django's Time Zone**

Django 包含了一个TIME_ZONE设置，默认为America/Chicago。这可能不是你居住的地方，因此你可能想要在你的设置文件中改变它。

## 映射URL到视图

重申一遍，这个视图函数返回一个包含当前日期和时间的HTML页面。要在一个特定的URL上显示该视图，你需要创建一个URLconf。查看[URL dispatcher](https://docs.djangoproject.com/en/1.6/topics/http/urls/) 的介绍。

## 返回错误

在Django中返回一个HTTP错误代码是很简单的。这里有一些HttpResponse的子类表示200(它意味着'OK')以外的一批常见的HTTP状态码。你可以在[request/response](https://docs.djangoproject.com/en/1.6/ref/request-response/#ref-httpresponse-subclasses)文档中找到可用子类的完整列表。只需返回那些子类的一个实例
来替代一个普通的HttpResponse来表示发生了错误。例如：

```python

from django.http import HttpResponse,HttpResponseNotFound

def my_view(request):
	# ...
	if foo:
		return HttpResponseNotFound('<h1>Page not found</h1>')
	else:
		return HttpResponse('<h1>Page was found</h1>')
```

没有为每个可能的HTTP响应代码定义专门的子类，因为它们很多都没有那么常见。然而，作为HttpResponse文档中的记录，你也可以传递HTTP状态码给HttpResponse构造器来为任何你喜欢的状态码创建一个返回类。例如：

```python

from django.http import HttpResponse

def my_view(request):
	# ...

	# Return a "created" (201) response code.
	return HttpResponse(status=201)
```

因为404错误是迄今为止最常见的HTTP错误，这里有一个简单的方式来错误这些错误。

### Http404 异常

**class django.http.Http404**

当你返回一个错误比如HttpResponseNotFound，你负责定义所得到的错误页面的HTML。

```python

return HttpResponseNotFound('<h1>Page not found</h1>')
```

为了方便起见，和在你的整个网站有一个一致的404错误页面是一个好主意，Django提供了一个Http404异常。如果你在一个视图函数的任意地方抛出Http404，Django将捕获它，并为你的应用返回一个标准的错误页面，并伴随着一个HTTP错误代码404。

示例用法：

```python 

from django.http import Http404
from django.shortcuts import render_to_response
from polls.models import Poll

def detail(request,poll_id):
	try:
		p = Poll.object.get(pk=poll_id)
	except Poll.DoesNotExist:
		raise Http404
	return render_to_response('polls/detail.html',{'poll':p})
```

为了充分利用Http404异常，你应该创建一个template，当抛出404错的时候，它就会被显示。这个template应该叫做404.html并且应该位于你模板树的顶层。

## 自定义错误视图

### 404 (网页不存在)视图

**django.views.defaults.page_not_found(request,template_name='404.html')**

当你从一个视图里抛出一个Http404,Django会加载一个特定的视图致力于处理404错误。默认，它是视图`django.views.defaults.page_not_found()`，它既产生一个非常简单的"Not Found"信息或者加载和呈现模板404.html，如果你在根模板目录里创建了它。

默认404视图将传递一个变量给模板：request_path，它是导致错误的URL。

这个page_not_found视图满足了99%的Web应用，但是如果你想要复写它，你可以在你的根URLconf(在其他任何地方设置handler404将不会有效果)中指定handler404，像这样：

```python

handler404 = 'mysite.views.my_custom_404_view'
```

在幕后，Django通过在你的根URLconf中查找handler404来决定404的视图，如果你没有定义，它就会回到django.views.defaults.page_not_found。

关于404视图有3件事需要注意：

- 如果Django在检查URLconf的每个正则表达式后没有发现一个匹配，404视图也会被调用。

- 传递一个RequestContext给404视图，它将会访问你的TEMPLATE_CONTEXT_PROCESSORS设置提供的变量。(如，MEDIA_URL)

- 如果DEBUG设置为True(在你的设置模块中)，那么你的404视图将永远不会使用到，将会显示你的URLconf和一些debug信息。

### 500 (服务器错误)视图

**django.views.defaults.server_error(request,template_name='500.html')**

同样，在视图代码中出现运行时错误的情况下，Django会执行特定的行为。如果一个视图产生了一个异常，默认的Django将会调用视图`django.views.defaults.server_error()`,它会产生一个非常简单的'Server Error'信息或者加载和呈现500.html模板，如果你在你的根模板目录里有创建它的话。

默认的500视图没有传递变量给500.html模板，用空的Context呈现以减少额外错误的机会。

这个server_error视图应该能满足99%的Web应用，但是如果你想要复写这个视图，你可以在你的根URLconf中指定handler500，像这样：

```python

handler500 = 'mysite.views.my_custom_error_view'
```

在幕后，Django通过在你的根URLconf中查找handler500来决定使用的500视图，如果你没有定义，则使用`django.views.defaults.server_error`。

关于500视图有一件事情需要注意：

- 如果DEBUG被设置为True(在你的设置模块中)，那么你的500视图不会被使用，而是会显示它的追踪信息和debug信息。

### 403 (HTTP 禁止)视图

**django.views.defaults.permission_denied(request,template_name='403.html')**

与404和500视图相同，Django有一个视图用于处理403禁止错误。如果一个视图产生403异常，那么Django会默认的调用视图`django.views.defaults.permission_denied`。

这个视图加载和呈现你根模板目录中的403.html模板，或者如果这个文件不存在，将使用'403 Forbidden'服务器文本替代，根据 [RFC 2616](http://tools.ietf.org/html/rfc2616.html)(HTTP 1.1 规范)。

`django.views.defaults.permission_denied`被PermissionDenied异常触发。你可以在你的视图代码中这样使用拒绝访问：

```python

from django.core.exceptions import PermissionDenied

def edit(request,pk):
	if not request.user.is_staff:
		raise PermissionDenied
	# ...
```

可以通过与你复写404和500视图相同的方式复写`django.views.defaults.permission_denied`，即在你的根URLconf中指定handler403：

```python

handler403 = 'mysite.views.my_custom_permission_denied_view'
```

### 400 (bad request)视图 

Django 1.6中新增。

**django.views.defaults.bad_request(request,template_name='400.html')**

当在Django中抛出SuspiciousOperation时，它会被Django的一个组件处理(例如，重置session数据)。如果没有特别的处理，Django将会考虑当前的请求是一个'bad request'，而不是一个服务器错误。

`django.views.defaults.bad_request`，在其他方面非常像server_error视图，但是它返回的是400状态码，来指示错误状态是一个客户端操作的结果。

像server_error，默认的bad_request应该可以满足99%的Web应用，但是如果你想要复写该视图，你可以在你的根URLconf中指定handler400，像这样：

```python

handler400 = 'mysite.views.my_custom_bad_request_view'
```

bad_request视图也仅用于DEBUG为False时。