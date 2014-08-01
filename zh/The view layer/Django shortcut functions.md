Django 快捷函数
=====================

django.shortcuts 包收集了辅助函数和跨MVC多层次的类。换句话说，为了方便起见这些函数和类引入了受控的连接。

## render

**render(request, template_name[, dictionary][, context_instance][, content_type][, status][, current_app])**

结合给定的context字典和给定的模板，返回带那个被呈现文本的HttpResponse对象。

`render()`与带context_instance调用`render_to_response()`一样，context_instance强制使用RequestContext。

Django不提供返回一个TemplateResponse的快捷函数，因为TemplateResponse的构造器提供了与`render()`相同水平的便利。

### 必填参数

- request

用于生成响应的请求对象。

- template_name

使用的模板的全名或者模板名称的序列。

### 可选参数

- dictionary

添加到模板上下文的值的字典。默认，它为空。如果字典中的一个值是可调用的，那么呈现给模板之前视图会调用它。

- context_instance

用于呈现模板的上下文实例。默认，模板将被一个RequestContext实例呈现(从request和字典中获得的值填充)。

- content_type

用于生成文档的MIME类型。默认为 [DEFAULT_CONTENT_TYPE](https://docs.djangoproject.com/en/1.6/ref/settings/#std:setting-DEFAULT_CONTENT_TYPE) 设置的值。

**Django 1.5中的变化**：

这个参数被称为mimetype。

- status

响应的状态码。默认为200。

- current_app

一个提示来指明包含当前视图的是哪个应用。查看 [namespaced URL resolution strategy](https://docs.djangoproject.com/en/1.6/topics/http/urls/#topics-http-reversing-url-namespaces) 了解更多信息。

### 例子

下面这个例子用MIME类型为application/xhtml+xml来呈现模板myapp/index.html。

```python

from django.shortcuts import render

def my_view(request):
	# View code here ...
	return render(request,'myapp/index.html',{'foo':'bar'},	content_type='application/xhtml+xml')
```

这个例子等于：

```python

from django.http import HttpResponse
from django.template import RequestContext,loader

def my_view(request):
	# View code here ...
	t = loader.get_template('myapp/index.html')
	c = RequestContext(request,{'foo':'bar'})
	return HttpResponse(t.render(c),content_type='application/xhtml+xml')
```

## render_to_response

**render_to_response(template_name[, dictionary][, context_instance][, content_type])**

以给定的上下文字典呈现给定的模板，并返回带被呈现文本的HttpResponse对象。

### 必填参数

- template_name

使用的模板的全名或者模板名称的序列。如果给定的是一个序列，第一个存在的模板将被使用。查看 [template loader documentation](https://docs.djangoproject.com/en/1.6/ref/templates/api/#ref-templates-api-the-python-api) 了解关于模板是怎么被发现的更多信息。

### 可选参数

- dictionary

添加到模板上下文的值的字典。默认，它为空。如果字典中的一个值是可调用的，那么呈现给模板之前视图会调用它。

- context_instance

用于呈现模板的上下文实例。默认，模板将被一个Context实例呈现(用从dictionary中的值填充)。如果你需要使用 [context processors](https://docs.djangoproject.com/en/1.6/ref/templates/api/#subclassing-context-requestcontext),使用RequestContext实例呈现代替。你的代码看起来像这样：

```python

return render_to_response('my_template.html',
                           my_data_dictionary,
                           context_instance=RequestContext(request))
```

- content_type

用于生成文档的MIME类型。默认为 [DEFAULT_CONTENT_TYPE](https://docs.djangoproject.com/en/1.6/ref/settings/#std:setting-DEFAULT_CONTENT_TYPE) 设置的值。

**Django 1.5中的变化**：

这个参数被称为mimetype。

### 例子

下面这个例子用MIME类型为application/xhtml+xml来呈现模板myapp/index.html。

```python

from django.shortcuts import render_to_response

def my_view(request):
	# View code here ...
	return render_to_response('myapp/index.html',{'foo':'bar'},content_type="application/xhtml+xml"))
```

这个例子等于：

```python

from django.http import HttpResponse
from django.template import Context,loader

def my_view(request):
	# View code here ...
	t = loader.get_template('myapp/index.html')
	c = Context({'foo':'bar'})
	return HttpResponse(t.render(c),content_type="application/xhtml+xml"))
```

## redirect

**redirect(to,[permanent=False,]\*args,\*\*kwargs)**

返回一个HttpResponseRedirect对象到合适的URL,并带传递的参数。

参数可以是：

- 一个model：模型的 `get_absolute_url()` 函数将被调用。

- 一个视图名称，可能带有参数：`urlresolvers.reverse` 将被用于反向解析这个名称。

- 一个URL，它将被按原样用于重定向定位。

### 例子

你可以以几种方式使用 `redirect()` 函数：

1、 传递一些对象；对象的 `get_absolute_url()` 方法将被调用来指出重定向的URL：
```python

from django.shortcuts import redirect

def my_view(request):
	...
	object = MyModel.objects.get(...)
	return redirect(object)
```
2、传递视图的名称和可选的位置或者关键字参数；将会使用 `reverse()` 方法来反向解析URL：
```python

def my_view(request):
	...
	return redirect('some-view-name',foo='bar') 
```
3、传递一个硬编码的URL来重定向：
```python

def my_view(request):
	...
	return redirect('/some/url/')
```

也可以传递全URL：
```python

def my_view(request):
	...
	return redirect('http://example.com/')
```

默认地，`redirect()` 返回一个临时的重定向。所有上面的这些形式接受一个permanent参数；如果设置为True，则会返回一个永久的重定向：
```python

def my_view(request):
	...
	object = MyModel.objects.get(...)
	return redirect(object,permanent=True)
```

## get_object_or_404

**get_object_or_404(klass,\*args,\*\*kwargs)**

在一个给定的model管理器上调用 `get()`，但是它抛出Http404而不是model的DoesNotExist异常。

### 必填参数

- klass

一个model类，一个管理器，或者一个获取对象的QuerySet实例。

- \*\*kwargs

查询参数，它应该是 `get()` 和 `filter()` 接受的格式。

### 例子

下面的例子从MyModel中获取主键为1的对象：
```python

from django.shortcuts import get_object_or_404

def my_view(request):
	my_object = get_object_or_404(MyModel,pk=1)
```

这个例子等于：
```python

from django.http import Http404

def my_view(request):
	try:
		my_object = MyModel.objects.get(pk=1)
	except MyModel.DoesNotExist:
		raise Http404
```

最常用的情况是传递一个Model,如上面展示的。然而，你也可以传递一个QuerySet实例：
```python

queryset = Book.objects.filter(title__startswith='M')
get_object_or_404(queryset,pk=1)
```

上面的例子有点的做作，因为它相当于：
```python

get_object_or_404(Book, title__startswith='M', pk=1)
```

但是如果你传递从地方得到的queryset变量，它也是有用的。

最后，你也可以使用一个管理器，如果你有一个[自定义管理器](https://docs.djangoproject.com/en/1.6/topics/db/managers/#custom-managers)，它也是有用的：
```python

get_object_or_404(Book.dahl_objects, title='Matilda')
```

你也可以使用相关的管理器：
```python

author = Author.objects.get(name='Roald Dahl')
get_object_or_404(author.book_set, title='Matilda')
```

注意：正如 `get()`,如果大于一个对象被发现，将会抛出MultipleObjectsReturned异常。

## get_list_or_404

**get_list_or_404(klass,\*args,\*\*kwargs)**

返回给定model管理器上filter()的结果并转换为一个列表，如果结果列表为空则抛出Http404。

### 必填参数

- klass 

一个Model,Manager或者一个获取列表的QuerySet实例。

- \*\*kwargs

查询参数，它应该是被 `get()` 和 `filter()`所接受的格式。

### 例子

下面的例子是从MyModel中获取所有已经发布的对象：
```python

from django.shortcuts import get_list_or_404

def my_view(request):
	my_objects = get_list_or_404(MyModel,published=True)
```

这个例子相当于：
```python

from django.http import Http404

def my_view(request):
	my_objects = list(MyModel.objects.filter(published=True))
	if not my_objects:
		raise Http404
```