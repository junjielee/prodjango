请求和响应对象
=================

## 简要概述

Django使用请求和响应对象通过系统来传递状态。

当请求一个页面，Django创建一个包含关于该请求的metadata的HttpRequest对象。那么Django加载适当的视图，以HttpRequest为第一个参数传递给视图函数。每个视图负责返回一个HttpResponse对象。

这个文档解释了HttpRequest和HttpResponse对象的API，它们是被定义在django.http模块中。

## HttpRequest对象

**class HttpRequest**

### 属性

所有属性应被认为是只读的，除非另有说明，下面的.session是个明显的例外。

**HttpRequest.body**

原HTTP请求body是一个byte字符串。用不同的方法处理数据比常规的HTML表单有用：二进制图片，XML负载等等。处理常规的表单数据，使用HttpRequest.POST。

你也可以用类文件接口从一个HttpRequest中读取。查看 [HttpRequest.read()](https://docs.djangoproject.com/en/1.6/ref/request-response/#django.http.HttpRequest.read) 。

**HttpRequest.path**

一个字符串表示请求页面的完整路径，不包含域名。

例如："/music/bands/the_beatles/"

**HttpRequest.path_info**

有一些Web服务器的配置，主机名后面的URL部分被分隔成一个脚本前缀部分和一个路径信息部分。path_info属性包含了路径的路径信息部分，无论使用的是什么Web服务器。使用它代替path可以是你的代码更容易在测试和开发服务器间迁移。

例如，你应用的WSGIScriptAlias被设置为"/minfo"，那么path可能是"/minfo/music/bands/the_beatles/"及path_info可能为"/music/bands/the_beatles/"。

**HttpRequest.method**

一个表示在请求中使用的HTTP方法的字符串。保证一定是大写。例如：
```python

if request.method == 'GET':
	do_something()
elif:request.method == 'POST':
	do_something_else()
```

**HttpRequest.encoding**

一个表示当前用于解码表单提交数据的编码的字符串(或者None，这意味着将会使用DEFAULT_CHARSET设置)。当访问表单数据的时候，你可以写这个属性来改变使用的编码。任何后续的属性访问(例如从GET或者POST中读)将会使用新的编码值。如果你知道表单数据不是DEFAULT_CHARSET编码时有用。

**HttpRequest.GET**

一个类字典对象包含了所有给出的HTTP GET参数。查看下面的 [QueryDict](https://docs.djangoproject.com/en/1.6/ref/request-response/#django.http.QueryDict) 文档。

**HttpRequest.POST**

一个类字典对象包含了所有给出的HTTP POST参数，提供请求包含的表单数据。查看下面的 [QueryDict](https://docs.djangoproject.com/en/1.6/ref/request-response/#django.http.QueryDict) 文档。如果你需要访问在请求中post的原生的或者没有表单数据，通过访问HttpRequest.body属性来代替。

**Django 1.5中的变化**：在Django 1.5之前，HttpRequest.POST中包含非表单数据。

一个请求有可能来自于带空POST字典的POST请求 - 如果，比如说，一个表单通过POST HTTP方法请求但是没有包含表单数据。因此，你不应该使用 `if request.POST` 来检测POST方法的使用；作为替代，可以使用 `if request.method == 'POST'` (见上文)。

注意：POST不包含文件上传信息。查看FILES。

**HttpRequest.REQUEST**

为了方便起见，一个类字典对象，它首先搜索POST，然后是GET。它的灵感来自于PHP的$_REQUEST。

例如，如果 `GET = {"name":"john"}` 和 `POST = {"age":'34'}` ，那么 `REQUEST["name"]` 将会是"john"，而 `REQUEST["age"]` 是"34"。

强烈建议使用GET和POST来代替REQUEST，因为前者更加明确。

**HttpRequest.COOKIES**

一个标准的Python字典包含了所有的cookies。键和值都是字符串。

**HttpRequest.FILES**

一个类字典的对象，它包含了所有上传的文件。FILES中的每个键是 `<input type="file" name"" />` 中的name。FILES中的每个值是下面描述的UploadedFile。

查看 [管理文件](https://docs.djangoproject.com/en/1.6/topics/files/) 了解更多信息。

需要注意的是如果请求的方法是POST并且post到请求的 `<form>` 有 `enctype="multipart/form-data"`,FILES只包含数据。否则FILES将是一个空的类字典的对象。

**HttpRequest.META**

一个标准的python字典包含了所有可用的HTTP头。可用的HTTP头依赖是客户端还是服务器，但是这里有一些例子。

  - CONTENT_LENGTH - 请求body的长度(以字符串表示)
  - CONTENT_TYPE - 请求body的MIME类型
  - HTTP_ACCEPT_ENCODING - 响应接受的编码
  - HTTP_ACCEPT_LANGUAGE - 响应接受的语言
  - HTTP_HOST - 由客户端发送的HTTP HOST 头
  - HTTP_REFERER - 引用页面，如果有的话
  - HTTP_USER_AGENT - 客户端的用户代理字符串
  - QUERY_STRING - 查询字符串，作为一个单个(未解析的)字符串
  - REMOTE_ADDR - 客户端的IP地址
  - REMOTE_HOST - 客户端的主机名
  - REMOTE_USER - 被WEB服务器认证过的用户，如果有的话。
  - REQUEST_METHOD - 一个字符串，如"GET"或者"POST"
  - SERVER_NAME - 服务器的主机名
  - SERVER_PORT - 服务器的端口(以字符串表示)

上面给出的，除了CONTENT_LENGTH和CONTENT_TYPE外，请求中的任何HTTP头被转换为META键，它通过转换所有字符为大写，用下划线替代任何的连字符并且在其名称上加上一个HTTP_前缀。因此，比如一个叫做X-Bender的头将会被映射到META键HTTP_X_BENDER上。

**HttpRequest.user**

一个AUTH_USER_MODEL类型的对象表示当前已登录的用户。如果用户当前未登录，user将被设置为 `django.contrib.auth.models.AnonymousUser` 的一个实例。你可以通过 `is_authenticated()` 来区分它们，像这样：
```python

if request.user.is_authenticated():
	# 为登录用户做点什么。
else:
	# 为匿名用户做点什么。
```

user仅在你的Django安装有AuthenticationMiddleware激活时可用。了解更多，请查看 [Django中的用户认证](https://docs.djangoproject.com/en/1.6/topics/auth/) 。

**HttpRequest.session**

一个可读写的，表示当前session的类字典对象。它仅在你的Django安装有session支持激活时可用。查看 [session文档](https://docs.djangoproject.com/en/1.6/topics/http/sessions/) 了解全部的细节。

**HttpRequest.urlconf**

Django本身没有定义，但是如果其他代码(例如，一个自定义的中间件类)有设置它，那么它将会被读取。如果存在，它将为当前请求用作根URLconf，覆盖ROOT_URLCONF设置。查看 [Django怎样处理一个请求](https://docs.djangoproject.com/en/1.6/topics/http/urls/#how-django-processes-a-request) 了解细节。

**HttpRequest.resolver_match**

Django 1.5 中新增。

它是ResolverMatch的一个实例来表示被解析的url。这个属性仅会在url解析发生后设置，这意味着它在所有视图中是可用的，但是在middleware方法中不可用，因为这些方法是在url解析发生前被执行(像process_request,你可以使用process_view代替)。

### 方法

**HttpRequest.get_host()**

返回请求的源主机地址，使用的信息是来自于 HTTP_X_FORWARDED_HOST(如果USE_X_FORWARDED_HOST被开启) 和 HTTP_HOST 头部，按照该顺序。如果他们没有提供一个值，方法会使用 SERVER_NAME 和 SERVER_PORT 的组合，详细的信息在[PEP 3333](http://www.python.org/dev/peps/pep-3333)中有描述。

例如："127.0.0.1:8000"

**注意**

当主机背后有多重代理时，`get_host()` 方法会失败。一种解决方法是使用middleware来重新代理头，如下面的例子：
```python

class MultipleProxyMiddleware(object):
	FORWARDED_FOR_FIELDS = [
	  'HTTP_X_FORWARDED_FOR',
	  'HTTP_X_FORWARDED_HOST',
	  'HTTP_X_FORWARDED_SERVER',
	]

	def process_request(self,request):
		"""
		重写代理头，使得仅有最近的代理被使用
		"""
		for field in self.FORWARED_FOR_FIELDS:
			if field in request.META:
				if ',' in request.META[field]:
					parts = request.META[field].split(',')
					request.META[field] = parts[-1].strip()
```

这个middleware应该定位在任何其他依赖 `get_host()` 值的middleware之前 - 例如，CommonMiddleware或者CsrfViewMiddleware。

**HttpRequest.get_full_path()**

返回path，附加上查询字符串，如果使用的话。

例如："/music/bands/the_beatles/?print=true"

**HttpRequest.build_absolute_uri(location)**

返回location的绝对URI形式。如果没有location提供，location将会被设置为 `request.get_full_path()` 。

如果location已经是一个绝对URI，那么它不会改变。否则将会使用这个请求中可用的服务器变量构建绝对URI。

例如："http://example.com/music/bands/the_beatles/?print=true"

**HttpRequest.get_signed_cookie(key,default=RAISE_ERROR,salt='',max_age=None)**

对于一个已签名的cookie返回一个cookie值，或者如果签名不再有效时，抛出 `django.core.signing.BadSignature` 异常。如果你提供了default参数，异常将会被异常并且作为代替default值会被返回。

可选参数salt参数用于提供额外的保护来防止对你的密钥的暴力攻击。如果提供max_age参数，那么它会被针对附加到cookie值的签名时间戳检查以确保cookie不会旧于max_age秒数。
```python

>>> request.get_signed_cookie('name')
'Tony'
>>> request.get_signed_cookie('name',salt='name-salt')
'Tony' # 假设cookie被使用相同的salt设置。
>>> request.get_signed_cookie('non-existing-cookie')
...
KeyError:'non-existing-cookie'
>>> request.get_signed_cookie('non-existing-cookie',False)
False
>>> request.get_signed_cookie('cookie-that-was-tampered-with')
...
BadSignature:...
>>> request.get_signed_cookie('name',max_age=60)
...
SignatureExpired:Signature age 1677.3839159 > 60 seconds
>>> request.get_signed_cookie('name',False,max_age=60)
False
```

查看 [cryptographic signing](https://docs.djangoproject.com/en/1.6/topics/signing/) 了解更多信息。

**HttpRequest.is_secure()**

如果请求是安全的，返回True。也就是说，如果它使用HTTPS。

**HttpRequest.is_ajax()**

如果请求是通过一个XMLHttpRequest生成，返回True，通过检查字符串'XMLHttpRequest'的 HTTP_X_REQUEST_WITH 头部。现在大多数JavaScript库都发送这个头部。如果你写你自己的XMLHttpRequest调用(在浏览器侧)，你将不得不手动设置这个头部，如果你想要 `is_ajax()` 工作的话。

如果一个响应在变化不论它是否通过AJAX请求的和你正使用某种形式的缓存像Django的cache_middleware，你应该使用 `vary_on_headers('HTTP_X_REQUEST_WITH')` 装饰视图，以使响应被正确缓存。

**HttpRequest.read(size=None)**

**HttpRequest.readline()**

**HttpRequest.readlines()**

**HttpRequest.xreadlines()**

**HttpRequest.\_\_iter\_\_()**

方法实现了一个类文件的接口来从一个HttpRequest实例中读取数据。这使得可以以流的方式处理一个传入的请求。一种常见的用法是以迭代解析器处理一个大的XML负载，而不需要在内存中构建整个XML树。

给定了这个标准的接口，可以直接传递一个HttpRequest实例给一个XML解析器,如ElementTree:
```python

import xml.etree.ElementTree as ET

for element in ET.iterparse(request):
	process(element)
```

## UploadedFile对象

**class UploadedFile**

### 属性

**UploadedFile.name**

上传文件的名称。

**UploadedFile.size**

上传文件的大小，单位是bytes 。

### 方法

**UploadedFile.chunks(chunk_size=None)**

返回一个产生连续的数据块的生成器。

**UploadedFile.read(num_bytes=None)**

从文件中读取字节数的数据。

## QueryDict对象

**class QueryDict**

在一个HttpRequest对象中，GET和POST属性是django.http.QueryDict的实例，它是一个类字典的定制类来处理相同的键有多个值的情况。这是很有必要的英文一些HTML表单元素，特别是( `<select multiple>` )，为相同的键传递多个值。

当在一个普通的request/response周期访问时，request.POST和request.GET中的QueryDict是不可变的。要获得一个可变版本你需要使用 `.copy()`。

### 方法

QueryDict实现了所有标准的字典方法，因为它是字典的一个子类。例外在这里列出：

**QueryDict.\_\_init\_\_(query_string,mutable=False,encoding=None)**

基于query_string实例化一个QueryDict对象：
```python

>>> QueryDict('a=1&a=2&c=3')
<QueryDict:{u'a':[u'1',u'2'],u'c':[u'1']}>
```

你遇到的大多数QueryDict,特别是那些在request.POST和QueryDict.GET中的，都是不可变的。如果你自己实例化一个，你可以通过传递 `mutable=True` 给他的\_\_init\_\_()来使它变为可变的。

如果encoding有设置为字符串，那么键和值将被从encoding转换为unicode。如果encoding没有设置，它默认为DEFAULT_CHARSET。

**QueryDict.\_\_getitem\_\_(key)**

返回给定键的值。如果键有超过一个值，那么\_\_getitem\_\_()会方法最新的值。如果键不存在，则抛出 `django.utils.datastructures.MultiValueDictKeyError` 异常(它是Python标准KeyError的一个子类，因此你也可以捕捉KeyError。)

**QueryDict.\_\_setitem\_\_(key,value)**

设置给定键到[value]中(一个Python列表，它的元素是value)。需要注意这点，作为其他字典函数有副作用，它仅能在可变QueryDict上调用(例如通过 `copy()` 创建的)。

**QueryDict.\_\_contains\_\_(key)**

如果给定的key存在返回True。这使得你可以这么做，例如，`if "foo" in request.GET`。

**QueryDict.get(key,default)**

使用上面的\_\_getitem\_\_()相同的逻辑，带一个hook使得如果key不存在返回一个default值。

**QueryDict.setdefault(key,default)**

像标准字典的 `setdefault()`方法，除了它使用的是内部的 `__setitem__()` 。

**QueryDict.update(other_dict)**

使用一个QueryDict或者标准的字典。就像使用标准字典的 `update()` 方法,除了它是附加到当前字典项目而不是替代它们。例如：
```python

>>> q = QueryDict('a=1',mutable=True)
>>> q.update({'a':'2'})
>>> q.getlist('a')
[u'1',u'2']
>>> q['a'] #返回最新的值
[u'2']
```

**QueryDict.items()**

就像标准字典的 `items()` 方法，除了，它使用与`__getitem__()` 相同的最新值逻辑。例如：
```python

>>> q = QueryDict('a=1&a=2&a=3')
>>> q.items()
[(u'a',u'3')]
```

**QueryDict.iteritems()**

就像标准字典的 `iteritems()` 方法。像 `QueryDict.items()` 一样它使用与`QueryDict.__getitem__()` 相同的最新值逻辑。

**QueryDict.iterlists()**

像 `QueryDict.iteritems()`,除了，它以列表的形式包含了字典的每个成员所有的值。

**QueryDict.values()**

像标准字典的 `values()` 方法，除了，它使用的是与 `__getitem__()`相同的最新值逻辑。例如：
```python

>>> q = QueryDict('a=1&a=2&a=3')
>>> q.values()
[u'3']
```

**QueryDict.itervalues()**

就像 `QuerDict.values()`,不过它是一个迭代器。

另外，QueryDict还有以下方法：

**QueryDict.copy()**

返回对象的一个副本，使用python标准库的 `copy.deepcopy()` 。这个副本是可变的，即使原对象不是。

**QueryDict.getlist(key,default)**

以一个python列表返回请求key的数据。如果key不存在及default值没有提供则返回一个空列表。除非default值不是列表，否则它保证返回一个某种排序的列表。

**QueryDict.setlist(key,list_)**

设置给定的key为list_(不像 `__setitem__()` )。

**QueryDict.appendlist(key,item)**

附加一个item到与key关联的内部的列表中。

**QueryDict.setlistdefault(key,default_list)**

就像setdefault,除了它使用的是一个值的列表而不是单个值。

**QueryDict.lists()**

就像 `items()`,除了它以列表的形式包含了字典每个元素的所有值。
```python

>>> q = QueryDict('a=1&a=2&a=3')
>>> q.lists()
[(u'a',[u'1',u'2',u'3'])]
```

**QueryDict.pop(key)**

返回key的值的列表并且从字典中删除它们。如果key不存在，抛出KeyError异常。
```python

>>> q = QueryDict('a=1&a=2&a=3',mutable=True)
>>> q.pop('a')
[u'1',u'2',u'3']
```

**QueryDict.popitem()**

删除字典的任意成员(没有顺序的概念)，并返回一个有两个值的元组包含了key和该key的所有值。如果对空字典调用该方法会抛出KeyErrory异常。例如：
```python

>>> q = QueryDict('a=1&a=2&a=3',mutable=True)
>>> q.popitem()
(u'a',[u'1',u'2',u'3'])
```

**QueryDict.dict()**

返回dict表示QueryDict。对于QueryDict中的每个(key,list),dict将有(key,item)，其中item是list的一个元素，使用 `QueryDict.__getitem__()` 一样的逻辑。
```python

>>> q = QueryDict('a=1&a=3&a=5')
>>> q.dict()
{u'a':u'5'}
```

**QueryDict.urlencode([safe])**

以查询字符串的格式返回一个数据的字符串。例如：
```python

>>> q = QueryDict('a=2&b=3&b=5')
>>> q.urlencode()
'a=2&b=3&b=5'
```

可选的，可以传递字符给urlencode，来不编码这些字符。例如：
```python

>>> q = QueryDict('',mutable=True)
>>> q['next'] = '/a&b/'
>>> q.urlencode(safe='/')
'next=/a%26b/'
```

## HttpResponse对象

**class HttpResponse**

与HttpRequest对象相反，它是被Django自动创建的，HttpResponse对象是你负责的。你写的每个视图是负责实例化，填充和返回一个HttpResponse。

HttpResponse类是在django.http模块中。

### 用法

#### 传递字符串

典型的用法是以字符串的形式传递页面的文本给HttpResponse构造器。
```python

>>> from django.http import HttpResponse
>>> response = HttpResponse("Here's the text of the Web page.")
>>> response = HttpResponse("Text only,please.",content_type="text/plain")
```

但是，如果你想增量的增加文本，你可以以类文件对象来使用response。
```python

>>> response = HttpResponse()
>>> response.write("<p>Here's the text of the Web page.</p>")
>>> response.write("<p>Here's another paragraph.</p>")
```

#### 传递迭代器

最后，你可以传递一个迭代器给HttpResponse而不是字符串。如果你使用这个技术，这个迭代器应该返回字符串。

传递一个迭代器作为内容给HttpResponse来创建一个流响应，当(且仅当)响应被返回之前没有middleware访问HttpResponse.content属性。

_Django 1.5中的变化_：

这个技术在Django 1.5中还不成熟，不推荐使用。如果你需要响应从迭代器到客户端的流式处理，你应该使用StreamingHttpResponse类代替。

作为Django 1.7，当HttpResponse被用迭代器实例化的时候，它将会立即使用它，以字符串储存响应内容，并丢弃迭代器。

_Django 1.5中的变化_：

你现在可以以类文件对象使用HttpResponse，即使它是被迭代器实例化的。Djang会在第一次访问的时候使用和保持迭代器的内容。

#### 设置头部字段

在你的响应中设置或者删除一个头部字段，可把它当作一个字典：
```python

>>> response = HttpResponse()
>>> response['Age'] = 120
>>> del response['Age']
```

需要注意的是，不像字典，如果头部字段不存在 `del` 不会抛出KeyError。

设置 Cache-Control 和 Vary 头部字段，建议使用来自django.utils.cache的 `patch_cache_control()` 和 `patch_vary_headers()` 方法，因为这些字段有多个，以逗号分隔的值。"patch"返回确保其他值，例如，被一个middleware添加的，不会被删除。

HTTP 头部字段不能包含新行。尝试设置一个头部字段包含一个新行字符(CR或者LF)将抛出BadHeaderError。

#### 告诉浏览器以文件附件处理响应

为了告诉浏览器以文件附件来处理一个响应，可以使用content_type参数和设置Content-Disposition头部。例如，这是你怎么样返回一个Microsoft的电子表格：
```python

>>> response = HttpResponse(my_data,content_type='application/vnd.ms-excel')
>>> response['Content-Disposition'] = 'attachment;filename="foo.xls"'
```

关于Content-Disposition头部没有Django-specific。但是容易忘记它的语法，因此我们在这里包含了它。

### 属性

**HttpResponse.content**

一个字符串表示内容，如果必要从一个Unicode对象编码。

**HttpResponse.status_code**

响应的[HTTP状态码](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10)。

**HttpResponse.reason_phrase**

_Django 1.6 中新增_

响应的HTTP原因短语。

**HttpResponse.streaming**

它一直为False。

此属性存在，因此，中间件可以处理streaming响应不同于常规响应。

### 方法

**HttpResponse.\_\_init\_\_(_content='',content_type=None,status=200,reason=None_)**

通过给定的页面内容和内容类型实例化一个HttpResponse对象。

*content*  应该为一个迭代器或者一个字符串。如果它是一个迭代器，它应该返回字符串，这些字符串将被连接在一起以形成响应的内容。如果它不是一个迭代器或者字符串，当被访问时，它将被转换为一个字符串。

*content_type*  是MIME类型，它为可选的。它由一个字符集编码和用于填充HTTP Content-Type头组成。如果没有指定，它由DEFAULT_CONTENT_TYPE 和 DEFAULT_CHARSET设置形成，默认为："text/html;charset=utf-8"。

从历史来看，这个参数被称为mimetype(现在已经过时了)。

*status*  为响应的[HTTP状态码](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10)。

_Django 1.6 中新增_

*reason*  是HTTP响应短语。如果没有提供，使用默认的短语。

**HttpResponse.\_\_setitem\_\_(_header,value_)**

设置header头部为value。header和value都为字符串。

**HttpResponse.\_\_delitem\_\_(_header_)**

删除给定名称的header。如果header不存在，静默失败。不区分大小写。

**HttpResponse.\_\_getitem\_\_(_header_)**

返回给定的头部名称的值。不区分大小写。

**HttpResponse.has_header(_header_)**

返回True或者False。它用于检测给定的头部是否存在。不区分大小写。

**HttpResponse.set_cookie(_key, value='', max_age=None, expires=None, path='/', domain=None, secure=None, httponly=False_)**

设置一个cookie。参数与python标准库中的Cookie.Morsel对象一样。

 - *max_age*  应该为秒数,如果cookie应该持续到客户端浏览器session时为None(默认)。如果*expires*没有指定，它将被计算。
 - *expires*  应该为以格式为"Wdy, DD-Mon-YY HH:MM:SS GMT"的字符串或者一个UTC时间的datetime.datetime对象。如果*expires*是一个datetime对象，*max_age*将被计算。
 - *domain*  如果你想要设置整个域的cookie，使用*domain*。例如，`domain=".lawrence.com"`将一个cookie，它可以被域名www.lawrence.com,blogs.lawrence.com,calendars.lawrence.com读取。否则，cookie仅被设置的域名读取。
 - 如果你想要防止客户端javaScript访问cookie，使用httponly。
   [HTTPOnly](https://www.owasp.org/index.php/HTTPOnly)是一个标志包含在一个设置cookie的HTTP响应头中。它不是cookie标准[RFC 2109](http://tools.ietf.org/html/rfc2109.html)的一部分，并且也不是所有的浏览器都支持。然而，当它被支持时，它就是一个减免客户端脚本访问受保护的cookie数据风险的很有用的方式。

**警告**

[RFC 2109](http://tools.ietf.org/html/rfc2109.html) 和 [RFC 6265](http://tools.ietf.org/html/rfc6265.html) 声明了用户代理应该至少支持4096字节的cookie。 对于很多浏览器这也是最大的值。如果尝试存储一个大于4096字节的cookie,Django不会抛出异常,但是很多浏览器不会正确的设置cookie。


**HttpResponse.set_signed_cookie(_key, value, salt='', max_age=None, expires=None, path='/', domain=None, secure=None, httponly=True_)**

像 `set_cookie()`,但是，在设置它之前 [加密签名](https://docs.djangoproject.com/en/1.6/topics/signing/) cookie。与 `HttpRequest.get_signed_cookie()` 配合使用。你可以使用可选参数salt来增加key的强度，但是你需要记住传递它给相应的 `HttpRequest.get_signed_cookie()` 调用。

**HttpResponse.delete_cookie(_key, path='/', domain=None_)**

使用给定的key删除cookie。如果key不存在，默默的失败。

由于cookie的工作方式，path与domain应该与你用于 `set_cookie()` 的是相同的值 - 否则cookie可能不会被删除。

**HttpResponse.write(_content_)**

这个方法使得一个HttpResponse实例像一个类文件的对象。

**HttpResponse.flush()**

这个方法使得一个HttpResponse实例像一个类文件的对象。

**HttpResponse.tell()**

这个方法使得一个HttpResponse实例像一个类文件的对象。

### HttpResponse子类

Django包含了一些HttpResponse的子类来处理不同类型的HTTP响应。跟HttpResponse一样，这些子类都在django.http中。

**class HttpResponseRedirect**

该构造器的第一个参数是必选的 - 重定向的路径。它可以是一个完整的URL(例如，'http://www.yahoo.com/search/')或者不带域名的绝对地址(例如，'/search/')。查看HttpResponse来了解构造器的其他可选参数。需要注意的是它返回一个HTTP状态码302。

  **url**

  _Django 1.6 中新增_

  这个只读属性表示响应将要重定向的URL(相当于Location响应头部)。

**class HttpResponsePermanentRedirect**

就像HttpResponseRedirect,但是它返回的是一个永久的重定向(HTTP状态码为300)来代替一个"found"的重定向(状态码为302)。

**class HttpResponseNotModified**

该构造函数不带任何参数，也不应该为该响应添加任何内容。使用这个来表示自从用户最后一次请求以来该页面还没有被修改过(状态码为304)。

**class HttpResponseBadRequest**

行为像HttpResponse，只不过使用状态码为400

**class HttpResponseNotFound**

行为像HttpResponse，只不过使用状态码为404

**class HttpResponseForbidden**

行为像HttpResponse，只不过使用状态码为403

**class HttpResponseNotAllowed**

行为像HttpResponse，只不过使用状态码为405。构造函数的第一个参数是必须的：为允许方法的列表。(例如，['GET','POST'])。

**class HttpResponseGone**

行为像HttpResponse，只不过使用状态码为410

**class HttpResponseServerError**

行为像HttpResponse，只不过使用状态码为500

## StreamingHttpResponse对象

_Django 1.5 中新增_

**class StreamingHttpResponse**

StreamingHttpResponse类用于stream一个Django响应到浏览器。如果生成响应需要花太多时间或者使用太多内存的话，你可能希望这么做。

**性能注意事项**

     Django是为短暂请求设计的。流响应将在整个响应时间内配合工作进程。这可能导致较差的性能。

     一般来说，你应该在请求-响应周期外执行消耗大的任务，而不是使用流响应。

StreamingHttpResponse类不是HttpResponse的子类，因为它采用了略有不同的API。然而，它几乎是相同的，但是有以下的值得注意的差异：

- 给它传递的是一个迭代器，产生的字符串作为内容。
- 你不能访问其内容，除了通过迭代响应对象本身。它只有在响应返回给客户端时发生。
- 它没有content属性。作为代替，它有一个streaming_content属性。
- 你不能使用类文件对象tell()或者write()方法。这样做会抛出一个异常。

StreamingHttpResponse应该仅用于绝对需要在传送数据给客户端之前整个内容不被迭代的情况下。因为内容不能被访问，许多中间件不能发挥功能。例如，不能为流响应生成ETag和Content-Length头部。

### 属性

**StreamingHttpResponse.streaming_content**

表示内容的一个字符串迭代器。

**StreamingHttpResponse.status_code**

响应的HTTP状态码。

**StreamingHttpResponse.reason_phrase**

_Django 1.6 中新增_

响应的HTTP原因短语。

**StreamingHttpResponse.streaming**

它总是为True。