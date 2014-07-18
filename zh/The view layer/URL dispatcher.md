URL dispatcher
=================

在高质量Web应用中，一个干净的、优雅的URL方案是一个很重要的细节。Django让你设计你想要的URL，没有框架限制。

没有.php或者.cgi要求，and certainly none of that 0,2097,1-1-1928,00 nonsense.

查看WWW的创建者Tim Berners-Len的 [Cool URIs don't change](http://www.w3.org/Provider/Style/URI) ，for excellent arguments on why URLs should be clean and usable.

## Overview

To design URLs for an app, you create a Python module informally called a URLconf (URL configuration). This module is pure Python code and is a simple mapping between URL patterns (simple regular expressions) to Python functions (your views).

This mapping can be as short or as long as needed. It can reference other mappings. And, because it’s pure Python code, it can be constructed dynamically.

Django also provides a way to translate URLs according to the active language. See the internationalization documentation for more information.

## How Django processes a request

When a user requests a page from your Django-powered site, this is the algorithm the system follows to determine which Python code to execute:

1. Django determines the root URLconf module to use. Ordinarily, this is the value of the ROOT_URLCONF setting, but if the incoming HttpRequest object has an attribute called urlconf (set by middleware request processing), its value will be used in place of the ROOT_URLCONF setting.
2. Django loads that Python module and looks for the variable urlpatterns. This should be a Python list, in the format returned by the function django.conf.urls.patterns().
3. Django runs through each URL pattern, in order, and stops at the first one that matches the requested URL.
4. Once one of the regexes matches, Django imports and calls the given view, which is a simple Python function (or a class based view). The view gets passed the following arguments:

	- An instance of HttpRequest.
	- If the matched regular expression returned no named groups, then the matches from the regular expression are provided as positional arguments.
	- The keyword arguments are made up of any named groups matched by the regular expression, overridden by any arguments specified in the optional kwargs argument to django.conf.urls.url().

5. If no regex matches, or if an exception is raised during any point in this process, Django invokes an appropriate error-handling view. See Error handling below.

## Example

这是一个URLconf的例子：

```python

from django.conf.urls import  patterns,url

urlpatterns = patterns('',
    url(r'^articles/2003/$','news.views.special_case_2003'),
    url(r'^articles/(\d{4})/$','news.views.year_archive'),
    url(r'^articles/(\d{4})/(\d{2})/$','news.views.month_archive'),
    url(r'^articles/(\d{4})/(\d{2})/(\d+)/$','news.views.article_detail'),
)
```

注意：

- 要从URL上抓取值，只需要把它放在括号里。
- 没有必要加一个前导斜杠，因为每个URL都有。例如,是^articles，而不是^/articles。
- 每个正则表达式字符串前面的'r'是可选的，但是推荐使用。它告诉Python这个字符串是"原生的" - 即是字符串中的任何字符都不需要转义。查看 [Dive Into Python’s explanation](http://www.diveintopython.net/regular_expressions/street_addresses.html#re.matching.2.3) 。

请求例子：

- 请求/articles/2005/03/将匹配列表中第3个条目。Django将调用函数 `news.views.month_archive(request,'2005','03')`。
- 请求/articles/2005/03/不会匹配任何URL模式，因为第3个条目要求月份为2个数字。
- 请求/articles/2003/将匹配列表中第一个模式，不是第二个，因为模式是按照顺序测试的，并且第1个第一次测试就通过了。可以像这样利用顺序插入特殊的案例。
- 请求/articles/2003 不会匹配任何的模式，因为每个模式要求URL以斜杠结尾。
- 请求/articles/2003/03/03/匹配最后一个模式。Django会调用函数 `news.views.articles_detail(request,'2003','03','03')`。

## 命名的组

上面的例子使用简单的，没有命名的正则表达式组(通过括号)来抓取URL的值并且以位置参数传递给一个视图。在一些更高级的用法中，有可能会使用命名的正则表达式组来抓取URL的值并以关键词参数传递给一个视图。

在Python正则表示中，命名的正则表达式组是 `(?P<name>pattern)`,这里的name是组的名称和pattern是要匹配的模式。

这里使用命名的组重写上面的例子：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^articles/2003/$','news.views.special_case_2003'),
    url(r'^articles/(?P<year>\d{4})/$','news.views.year_archive'),
    url(r'^articles/(?P<year>\d{4})/(?P<month>\d{2})/$','news.views.month_archive'),
    url(r'^articles/(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/$','news.views.article_detail'),
)
```

这跟上面的例子正好是完成同样的事情，有个微妙的差异是：抓取的值是通过关键词参数传递给视图，而不是位置参数。例如：

- 请求/articles/2005/03/将调用函数 `news.views.month_archive(request,year='2005',month='03')`,而不是`news.views.month_archive(request,'2005','03')`。
- 请求/articles/2005/03/03/将调用函数 `news.views.month_archive(request,year='2005',month='03',day='03')`。

在实践中，这意味着你的URLconfs更加的清晰，并且不易发生参数顺序bug - 并且你可以在你的视图函数定义中重新排列参数。当然，它的优势在于简洁。一些开发者认为命名的组语法丑陋并且冗长。

### 匹配/分组算法

URLconf 解析器的算法如下，针对正则表达式中命名组和非命名组：

1. 如果有任何的命名参数，则会使用它们，并且忽略非命名的参数。
2. 否则，它将以位置参数传递所有非命名参数。

在这两种情况中，任何额外的[ Passing extra options to view functions](https://docs.djangoproject.com/en/1.6/topics/http/urls/#passing-extra-options-to-view-functions) 中给出的关键词参数也会被传递给视图。

## URLconf 搜索什么

URLconf 以普通的Python字符串搜索一个请求URL。它不包括GET或者POST参数，或者域名。

例如，请求 http://www.example.com/myapp/，URLconf将查找myapp/。

请求 http://www.example.com/myapp/?page=3，URLconf将查找myapp/。

URLconf 不看请求方法。换句话说，任何的请求方法 - POST、GET、HEAD等等。 - 相同的URL都会被路由到相同的函数。

## 抓获的参数始终是字符串

每个被抓获的参数是以Python字符串被发送到视图的，无论使用的哪种正则表达式匹配产生的。例如，这个URLconf:

```python

url(r'^articles/(?P<year>\d{4})/$', 'news.views.year_archive'),
```

传递给 `news.views.year_archive()` 的year参数是字符串，不是整数，虽然`\d{4}`仅匹配整数字符串。

## 为视图参数指定默认的值

一个方便的技巧是为你的视图参数指定默认的值。这里有一个URLconf和视图的例子：

```python

# URLconf
from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^blog/$', 'blog.views.page'),
    url(r'^blog/page(?P<num>\d+)/$', 'blog.views.page'),
)

# View (in blog/views.py)
def page(request, num="1"):
    # 根据num，输出相应的博客页面
    ...
```

上面的例子，两个URL模式都指向相同的视图 - blog.views.page - 但是第一个模式不从URL抓获任何的值。如果匹配到第一个模式，`page()`函数将使用参数的默认值。如果第二个参数匹配，`page()`将使用通过正则表达式抓获的num值。

## 性能

urlpatterns中的每个正则表达式，在第一次访问时被编译。这使得系统速度极快。

## 变量urlpatterns的语法

urlpatterns是一个以函数`django.conf.urls.patterns()`返回的格式的Python列表。常使用`patterns()`创建urlpatterns变量。

## 错误处理

当Django没有发现一个正则表达式来匹配请求的URL，或者当抛出一个异常，Django将调用错误处理视图。

用于这些案例的视图有四个变量指定。其默认值满足大多数工程的需求，但是可通过为它们赋值来进一步的自定义。

查看 [customizing error views](https://docs.djangoproject.com/en/1.6/topics/http/views/#customizing-error-views) 文档，了解全部的细节。

这样的值可以在你的根URLconf中设置。在其他任何的URLconf中设置这些变量没有任何效果。

这些值必须是可调用的，或者是表示视图的完整的Python导入路径的字符串，这个视图被调用来处理错误条件。

这些变量是：

- handler404 - 查看 [django.conf.urls.handler404](https://docs.djangoproject.com/en/1.6/ref/urls/#django.conf.urls.handler404)
- handler500 - 查看 [django.conf.urls.handler500](https://docs.djangoproject.com/en/1.6/ref/urls/#django.conf.urls.handler500)
- handler403 - 查看 [django.conf.urls.handler403](https://docs.djangoproject.com/en/1.6/ref/urls/#django.conf.urls.handler403)
- handler400 - 查看 [django.conf.urls.handler400](https://docs.djangoproject.com/en/1.6/ref/urls/#django.conf.urls.handler400)

## 视图前缀

你可以在你的`patterns()`调用中指定一个通用的前缀，以减少重复的代码。

这是 [Django overview](https://docs.djangoproject.com/en/1.6/intro/overview/) 中的示例URLconf。

```python

from django.conf.urls import patterns,url

urlpatterns = patterns('',
    url(r'^articles/(\d{4})/$', 'news.views.year_archive'),
    url(r'^articles/(\d{4})/(\d{2})/$', 'news.views.month_archive'),
    url(r'^articles/(\d{4})/(\d{2})/(\d+)/$', 'news.views.article_detail'),
)
```

上面这个例子，每个视图都有一个通用的前缀 - 'news.views'。代替在urlpatterns中每个条目中指出来，你可以在函数`patterns()`中使用第一个参数来为每个视图函数指定一个通用的前缀。

考虑到这点，上面的例子可以简洁的写为：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('news.views',
    url(r'^articles/(\d{4})/$', 'year_archive'),
    url(r'^articles/(\d{4})/(\d{2})/$', 'month_archive'),
    url(r'^articles/(\d{4})/(\d{2})/(\d+)/$', 'article_detail'),
)
```

需要注意的是，你不需要在前缀的后面放点(.)。Django会为你自动添加。

### 多视图前缀

在实践中，你很可能会最终混合和匹配指向urlpatterns中没有通用前缀的视图。然而你仍然可以使用视图前缀的快捷的好处来消除重复。只需要添加多个`patterns()`对象在一起，像这样：

旧：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^$', 'myapp.views.app_index'),
    url(r'^(?P<year>\d{4})/(?P<month>[a-z]{3})/$', 'myapp.views.month_display'),
    url(r'^tag/(?P<tag>\w+)/$', 'weblog.views.tag'),
)
```

新：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('myapp.views',
    url(r'^$', 'app_index'),
    url(r'^(?P<year>\d{4})/(?P<month>[a-z]{3})/$','month_display'),
)

urlpatterns += patterns('weblog.views',
    url(r'^tag/(?P<tag>\w+)/$', 'tag'),
)
```

## 介绍其他URLconf

在任何点，你的urlpatterns可以'include'其他URLconf模块。这实质上是包含了'根'以下的其他URLs的组。

例如，这是Django Web站本身的URLconf摘录。它包含了一些其他的URLconf:

```python

from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    # ... snip ...
    url(r'^comments/', include('django.contrib.comments.urls')),
    url(r'^community/', include('django_website.aggregator.urls')),
    url(r'^contact/', include('django_website.contact.urls')),
    # ... snip ...
)
```

需要注意的是这个例子中的正则表达式没有$(结束的字符串匹配字符)，但是包含结尾的斜杆。当Django碰到`include()`( `django.conf.urls.include()` )，它会砍掉到此为止匹配到的URL部分并且发送剩余的字符串到include的URLconf作进一步的处理。

另一种可能是，包含了不是通过URLconf Python模块指定的，而是以'include()'参数定义的，通过直接使用`patterns()`返回的模式列表指定的额外的URL模式。例如，考虑这个URLconf：

```python

from django.conf.urls import include, patterns, url

extra_patterns = patterns('',
    url(r'^reports/(?P<id>\d+)/$', 'credit.views.report'),
    url(r'^charge/$', 'credit.views.charge'),
)

urlpatterns = patterns('',
    url(r'^$', 'apps.main.views.homepage'),
    url(r'^help/', include('apps.help.urls')),
    url(r'^credit/', include(extra_patterns)),
)
```

在这个例子中，/credit/reports/ URL将被 `credit.views.report()` Django 视图处理。

这个可用于从一个单一模式前缀被重复使用的URLconf中消除冗余。例如，考虑这个URLconf:

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('wiki.views',
    url(r'^(?P<page_slug>\w+)-(?P<page_id>\w+)/history/$', 'history'),
    url(r'^(?P<page_slug>\w+)-(?P<page_id>\w+)/edit/$', 'edit'),
    url(r'^(?P<page_slug>\w+)-(?P<page_id>\w+)/discuss/$', 'discuss'),
    url(r'^(?P<page_slug>\w+)-(?P<page_id>\w+)/permissions/$', 'permissions'),
)
```

我们可以通过使用一次通用路径前缀并分组不同的后缀来提高它：

```python

from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    url(r'^(?P<page_slug>\w+)-(?P<page_id>\w+)/', include(patterns('wiki.views',
        url(r'^history/$', 'history'),
        url(r'^edit/$', 'edit'),
        url(r'^discuss/$', 'discuss'),
        url(r'^permissions/$', 'permissions'),
    ))),
)
```

### 捕获的参数

一个包含的URLconf接受任意从父URLconf捕获的参数,因此下面的例子是有效的：

```python

# In settings/urls/main.py
from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    url(r'^(?P<username>\w+)/blog/', include('foo.urls.blog')),
)

# In foo/urls/blog.py
from django.conf.urls import patterns, url

urlpatterns = patterns('foo.views',
    url(r'^$', 'blog.index'),
    url(r'^archive/$', 'blog.archive'),
)
```

上面的例子中，捕获的"username"变量会被如期望的传递到包含的URLconf中。

## 传递额外的选项给视图函数

URLconf 有一个钩子让你以一个Python字典的形式传递额外的参数给你的视图函数。

`django.conf.urls.url()` 函数接受一个以字典作为可选的第三个参数，并且会将它作为额外的关键词参数来传递给视图函数。

例如：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('blog.views',
    url(r'^blog/(?P<year>\d{4})/$', 'year_archive', {'foo': 'bar'}),
)
```

在这个例子中，请求 /blog/2005/，Django将调用 `blog.views.year_archive(request,year='2005',foo='bar')` 。

这个技术用于 [syndication framework](https://docs.djangoproject.com/en/1.6/ref/contrib/syndication/) 传递metadata和选项给视图。

** 处理冲突 **

有可能有一个URL模式捕获到命名的关键词参数与字典中传递的额外参数有相同的名称。当这个发生时，字典中的参数将被用于代替在URL中捕获到的参数。

### 传递额外的选项给include()

相似的，你可以传递额外的选项给 `include()`。当你传递额外的选项给 `include()` 时，每行中被include的URLconf会被传递额外的选项。

例如，这两个URLconf集在功能上是一样的：

Set one :

```python

# main.py
from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    url(r'^blog/', include('inner'), {'blogid': 3}),
)

# inner.py
from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^archive/$', 'mysite.views.archive'),
    url(r'^about/$', 'mysite.views.about'),
)
```

Set two:

```python

# main.py
from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    url(r'^blog/', include('inner')),
)

# inner.py
from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^archive/$', 'mysite.views.archive', {'blogid': 3}),
    url(r'^about/$', 'mysite.views.about', {'blogid': 3}),
)
```

需要注意的是，额外的选项常会传递给被包含的URLconf的每行，不管该行视图是否实际有效的接收这些选项。为此原因，该技术仅对如果你确信被包含的URLconf的每个视图都接受你传递的额外选项有用。

## 传递可调用对象代替字符串

一些开发者发现传递实际的Python函数对象而不是包含它模块路径的字符串更自然。支持这个替代 - 你可以传递任何的可调用对象作为视图。

例如，用"字符串"符号给出URLconf:

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^archive/$', 'mysite.views.archive'),
    url(r'^about/$', 'mysite.views.about'),
    url(r'^contact/$', 'mysite.views.contact'),
)
```

你可以通过传递对象而不是字符串来完成相同的事情。只是一定要导入对象：

```python

from django.conf.urls import patterns, url
from mysite.views import archive, about, contact

urlpatterns = patterns('',
    url(r'^archive/$', archive),
    url(r'^about/$', about),
    url(r'^contact/$', contact),
)
```

下面的例子在功能上是相同的。它只是更加紧凑，因为它导入包含views的模块，而不是单独导入每个视图：

```python

from django.conf.urls import patterns, url
from mysite import views

urlpatterns = patterns('',
    url(r'^archive/$', views.archive),
    url(r'^about/$', views.about),
    url(r'^contact/$', views.contact),
)
```

使用哪种风格取决于你自己。

需要注意的是如果你使用了这项技术 - 传递对象而不是字符串 - 视图前缀(上面"视图前缀"说解释的)将无效。

需要注意的是 [基于类的视图](https://docs.djangoproject.com/en/1.6/topics/class-based-views/) 必须要被导入:

```python

from django.conf.urls import patterns, url
from mysite.views import ClassBasedView

urlpatterns = patterns('',
    url(r'^myview/$', ClassBasedView.as_view()),
)
```


## URL的反向解析

在Django中工作时，一个常见的需求是在它们的最后形式中包含URL，无论是嵌入到生成的内容(视图和URL资源，或者显示给用户的URL，等等)或者在服务器端导航流的处理(重定向，等等。)的可能性。

我们强烈的希望不要硬编码这些URL(一个费力的，不可扩展的，容易出错的策略)或者不得不为生成平行于URLconf描述设计的URL制定点对点机制。正因为如此，在某些时候有产生陈旧URL的分享。

换句话说，我们需要的是DRY机制。除其他优点以外，它允许URL设计的演进，而不必在所有项目的源代码中搜索和替代过时的URL。

我们可用的作为一个开始点(a starting point)以获得一个URL的信息片段是负责处理它的视图的一个标识符(比如，名称)，必要的参与查找正确URL的其他信息片段是视图参数的类型(位置，关键词)和值。

Django提供了一个解决方案，使得URL映射器是唯一的URL设计的储存库。你可以用URLconf配置它，它用于两个方向：

- 以用户/浏览器请求的URL开始，它将调用正确的Django视图，并提供它可能需要的任意参数与从URL中捕获的值。
- 以相应的Django视图加上传递给它的参数值作为标识符开始，获得相关的URL。

第一个的用法我们已经在前面的章节讨论过了。第二个就是所谓的URL的方向解析，反向URL匹配，反向URL查找，或者简单的URL反向：

Django提供了用于执行URL反向的，匹配需要URL的不同层的工具：

- 在模板中：使用url 模板标签。
- 在Python代码中：使用 `django.core.urlresolvers.reverse()` 函数。
- 在相关的Django模型实例的URL的处理的更高层代码中： `get_absolute_url()` 方法。

### 例子

再次考虑这个URLconf条目：

```python

from django.conf.urls import patterns, url

urlpatterns = patterns('',
    #...
    url(r'^articles/(\d{4})/$', 'news.views.year_archive'),
    #...
)
```

根据这个设计，nnnn年归档对应的URL是 /articles/nnnn/ 。

你可以在模板代码中包含它们，使用：

```HTML

<a href="{% url 'news.views.year_archive' 2012 %}">2012 Archive</a>
{# Or with the year in a template context variable: #}
<ul>
{% for yearvar in year_list %}
<li><a href="{% url 'news.views.year_archive' yearvar %}">{{ yearvar }} Archive</a></li>
{% endfor %}
</ul>
```

或者在Python代码中：

```python

from django.core.urlresolvers import reverse
from django.http import HttpResponseRedirect

def redirect_to_year(request):
    # ...
    year = 2006
    # ...
    return HttpResponseRedirect(reverse('news.views.year_archive', args=(year,)))
```

如果由于有些原因，决定要改变文章按年归档发布的URL，那么你仅需要改变URLconf的条目。

在某些情况下，视图都是一般类型的，在URL和视图间可能存在一个多对一的关系。对于这种情况，当它涉及到反向解析URL时，视图名称不是一个足够好的标识符。阅读下一章了解Django提供的关于这点的解决方案。

## Naming URL patterns


## URL namespaces

### Introduction
### Reversing namespaced URLs
### URL namespaces and included URLconfs

#### Example