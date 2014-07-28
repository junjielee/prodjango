URL 调度器
=================

在高质量Web应用中，一个干净的、优雅的URL方案是一个很重要的细节。Django让你设计你想要的URL，没有框架限制。

没有.php或者.cgi要求，当然没有 0,2097,1-1-1928,00 废话。

查看WWW的创建者Tim Berners-Len的 [Cool URIs don't change](http://www.w3.org/Provider/Style/URI)，这是为什么URL应该是干净并且可用的的良好论据。

## 概要

为一个app设计URL，你创建一个Python模块非正式的称为一个URLconf(URL配置)。这个模块是纯Python代码，它是一个简单的URL模式(简单的正则表达式)与Python函数(你的视图)间的映射。

这个映射可长可短，这依赖你的需求。它可以引用其他映射。并且，因为它是纯Python代码，它可以被动态的构造。

Django 还提供了根据当前语言来翻译URL的方法。查看[internationalization documentation](https://docs.djangoproject.com/en/1.6/topics/i18n/translation/#url-internationalization)了解更多信息。

## Django怎样处理一个请求

当一个用户请求你的Django驱动的站点的一个页面时，如下是该系统的算法，来确定哪些代码被执行：

1. Django确定根URLconf模块来使用。 按理说，它是 ROOT_URLCONF 设置的值，但是如果传入的HttpRequest对象有一个名为URLconf的属性(通过中间件请求处理设置)，那么它的值将被使用来代替 ROOT_URLCONF 设置。

2. Django加载Python模块和查找urlpatterns变量。它应该是一个以函数 `django.conf.urls.patterns()` 返回的格式Python列表，

3. Django 按顺序运行每个URL模式，并停止于匹配请求URL的第一个模式。

4. 一旦一个正则表达式匹配，Django会导入和调用给定的视图，它是一个简单的Python函数(或者基于视图的类)。视图被传递以下参数：

	- HttpRequest的一个实例。

	- 如果匹配的正则表达式返回没有命名的组，那么正则表达式匹配项将会以位置参数被提供。

	- 被正则表达式匹配的任何命名组都会生成关键字参数，它会被传递给 `django.conf.urls.url()` 的可选关键字参数中指定的参数覆盖。

5. 如果没有正则表达式匹配，或者如果在这个过程中的任意点抛出一个异常，Django会调用一个适当的错误处理视图。查看下面的错误处理。

## 例子

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

## 命名URL模式

在URLconf中，多个URL模式使用相同的函数是非常常见的。例如，这两个URL模式都指向archive视图：

```python

from django.conf.urls import patterns, url
from mysite.views import archive

urlpatterns = patterns('',
    url(r'^archive/(\d{4})/$', archive),
    url(r'^archive-summary/(\d{4})/$', archive, {'summary': True}),
)
```

这完全的是有效的，但是当你尝试去反向URL匹配时(通过`reverse()函数或者url模板标签`)会导致一些问题。继续看这个例子，如果你想获取archive视图的URL，Django反向URL匹配就会产生困惑，因为两个URL模式指向这个视图。

为了解决这个问题，Django支持命名的URL模式。即就是，你可以为一个URL模式指定一个名称来与其他使用相同视图和参数的URL模式区分开。然后，你就可以在反向URL匹配中使用该名称了。

这里是上面的例子使用命名的URL模式重写的：

```python

from django.conf.urls import patterns, url
from mysite.views import archive

urlpatterns = patterns('',
    url(r'^archive/(\d{4})/$', archive, name="full-archive"),
    url(r'^archive-summary/(\d{4})/$', archive, {'summary': True}, name="arch-summary"),
)
```

有了这些名字(full-archive和arch-summary)，您可以通过使用其名称分别定位每个模式：

```html

{% url 'arch-summary' 1945 %}
{% url 'full-archive' 2007 %}
```

即使这里两个URL模式都指向archive视图，在django.conf.urls.url()中使用name参数可以让你在模板中把他们区分开来。

用于URL名称的字符串可以包含你喜欢的任意字符。不仅限于有效的Python名称。

**注意**

当你命名你的URL模式，请确保您使用的名字不太可能与任何其他应用程序所选择的名称发生冲突。如果你调用你的URL模式comment，另一个应用也做相同的事情，当你使用这个名称的时候，不能保证哪个URL将被插入到你的模板中。

为你的URL名称加上一个前缀，也许来源于应用的名称，将减少发生冲突的机会。我们推荐使用myapp-comment代替comment。

## URL命名空间

### 介绍

当你需要部署单个应用的多个实例时，能区分不同的实例是非常有用的。当使用命名的URL模式的时候，这点特别的重要，因为单个应该的多个实例将分享命名的URL。命名空间提供了一种来分辨这些命名的URL的方法。

一个URL的命名空间有两部分，他们都是字符串：

**应用命名空间**

它描述了正被部署的应用的名称。应用的每个实例都有相同的应用命名空间。例如，Django的admin应用有些可预见的应用命名空间'admin'。

**实例命名空间**

它标识了一个应用特定的实例。实例命名空间应该在你的整个工程中是唯一的。然而，一个实例命名空间可以与应用命名空间一样。它用于为一个应用指定一个默认的实例。例如，默认的Django Admin实例有一个'admin'的实例命名空间。

命名空间的URL被使用':'操作符指定。例如，admin应用的主页用admin:index引用。它表明了一个命名空间'admin'和一个命名的URL'index'。

命名空间也可以嵌套。命名URL'foo:bar:whiz'将在命名空间'bar'中查找名称为'whiz'的模式，'bar'为在顶级命名空间'foo'中定义的。

### 反向解析命名空间URL

当给出一个命名空间URL(例如，'myapp:index')来解析，Django会分割完整的名称，然后尝试一下查询：

 1. 第一，Django查找匹配的应用命名空间(在这个例子中，'myapp')。它将产生该应用实例的列表。

 2. 如果有定义当前应用，Djang查找并返回该实例的URL。当前应用可以被指定为模板上下文上的一个属性 - 期望有多个部署的应用应该在任何用于呈现一个模板的Context或者RequestContext上设置current_app属性。
 当前应用也可以通过 `django.core.urlresolvers.reverse()` 函数的一个参数手动指定。

 3. 如果没有当前应用。Django会查找一个默认应用实例。默认应用实例是有一个实例命名空间匹配应用命名空间(在这个例子中，myapp的一个实例叫'myapp')的实例。

 4. 如果没有默认应用实例，Django将挑最新部署的应用实例，无论它的实例名称是什么。

 5. 如果在步骤1中，提供的命名空间没有匹配应用命名空间，Django将试图将这个命名空间作为实例命名空间直接查找。

如果有嵌套的命名空间，命名空间的每个部分都会重复这些步骤，直到只剩下视图名未解析。然后，视图名称将会被解析为被发现的命名空间中的一个URL。

#### 例子

为了展示这个解决策略，考虑myapp的两个实例的例子：一个叫做'foo'，另一个叫做'bar'。myapp有一个主页URL命名为'index'。使用这个设置，以下的查询是可能的：
 - 如果一个实例是当前的 - 就是说，如果我们在实例'bar'中呈现一个实用页面 - 'myapp:index'将会解析为实例'bar'的主页。

 - 如果没有当前实例 - 就是说，如果我们在这个网站上的其他地方展示一个页面 - 'myapp:index'将解析为myapp最新注册的实例。因为没有默认的实例，myapp的最新的实例将被使用。可能是'foo'或者'bar',这取决与他们被引入工程的urlpattern的顺序。

 - 'foo:index'总是会被解析为实例'foo'的主页。

如果也有一个默认的实例 - 即，名称为'myapp'的实例 - 会发生如下的事情：

 - 如果一个实例是当前的 - 就是说，如果我们在实例'bar'中呈现一个实用页面 - 'myapp:index'将会解析为实例'bar'的主页。

 - 如果没有当前实例 - 就是说，如果我们在这个网站上的其他地方展示一个页面 - 'myapp:index'将解析为默认实例的主页。

 - 'foo:index'还是会解析为实例'foo'的主页。

### URL 命名空间和包含 URLconf 

包含的URLconf的URL命名空间可以通过两种方式指定。

首先，你可以通过在你构造你的URL模式的时候为 `django.conf.urls.include()` 提供应用和实例命名空间作为参数。例如：

```python

url(r'^help/', include('apps.help.urls', namespace='foo', app_name='bar')),
```

这将包含定义在apps.help.urls中，应用命名空间为'bar'，实例命名空间为'foo'的URL。

第二，你可以包含嵌入式的命名空间数据的对象。如果你 `include()` 一个被 `patterns()` 返回的对象，包含在这个对象中的URL会被添加到全局命名空间。然而，你也可以 `include()` 一个3-元组包含：

```python

(<patterns object>, <application namespace>, <instance namespace>)
```

例如：

```python

from django.conf.urls import include, patterns, url

help_patterns = patterns('',
    url(r'^basic/$', 'apps.help.views.views.basic'),
    url(r'^advanced/$', 'apps.help.views.views.advanced'),
)

url(r'^help/', include((help_patterns, 'bar', 'foo'))),
```

它将包含提名的URL模式到给定的应用和实例命名空间。

例如，Django Admin被部署为AdminSite的一个实例。AdminSite对象有一个urls属性：一个包含了相应的admin站点中的所有模式，加上应用命名空间'admin'，和admin实例名的3元素元组。当你部署一个Admin实例时，正是这个urls属性被你 `include()` 到你的工程的urlpatterns中。

务必要传递一个元组给 `include()`。如果你简单的传递3个参数：`include(help_patterns,'bar','foo')`,Django不会抛出错误，但是由于 `include()`的签名，'bar'将是实例命名空间和'foo'将是应用命名空间，而不是反之亦然。