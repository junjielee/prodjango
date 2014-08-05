内置视图
===========

Django的内置视图记录在了[编写视图](https://docs.djangoproject.com/en/1.6/topics/http/views/)以及文档的其他地方。

## 在开发中serve文件

**static.serve(request, path, document_root, show_indexes=False)**

可能有些文件不是你工程的静态资源，为了方便起见，在本地开发中，你可能想要Django为你服务。`serve()`视图能用于服务任何你给出的目录。（这个视图不是硬化为生产使用，它应仅用作辅助开发；在生产环境中你应该使用一个真正的前端web服务器服务这些文件）。

最有可能的实例是在MEDIA_ROOT中的用户上传的内容。django.contrib.staticfiles 适用于静态资源，对于用户上传的文件没有内置的处理，但是你可以通过在你的URLconf后面追加一些东西来让Django 服务你的MEDIA_ROOT，就像这样：
```python

from django.conf import settings

# ... 你的URLconf的其余部分放在这里 ...

if settings.DEBUG:
	urlpatterns += patterns('',
		url(r'^media/(?P<path>.*)$',
		    'django.views.static.serve',
		    {'document_root':settings.MEDIA_ROOT},
		   ),
		)
```

需要注意的是，该代码假设你的MEDIA_URL有'/media/'的一个值。这将会调用 `serve()`视图，传递从URLconf中获得的path和(必选的)document_root参数。

由于这样定义这个URL模式有点麻烦，Django自带了一个小的URL辅助函数 [static()](https://docs.djangoproject.com/en/1.6/ref/urls/#django.conf.urls.static.static),它以prefix和view为参数，例如prefix为MEDIA_URL,和view为'django.views.static.serve'。其他任何的函数参数将透明地传递给视图。