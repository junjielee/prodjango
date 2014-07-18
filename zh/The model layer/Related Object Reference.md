Related object reference
===========================

- class RelatedManager

"related manager"是用于管理one-to-many 或者 many-to-many相关的上下文。这发生在两种情况下：

  - 外键关系的"另一方"，即就是：

```python

from django.db import models

class Reporter(models.Model):
    #...
    pass
class Article(models.Model):
  reporter = models.ForeignKey(Reporter)
```

在上述的例子中，下面的方法在reporter.article_set上可用。

  - ManyToManyField关系的两边：

```python
	 
class Topping(models.Model):
    # ...
    pass

class Pizza(models.Model):
    toppings = models.ManyToManyField(Topping)
```

在这个例子中，下面的方法在topping.pizza_set和pizz.toppings上同时可用。

这些related managers有一些额外的方法：

  - add(obj1[,obj2,...])

添加指定的model对象到相关的对象集中。

例如：

```python

>>> b = Blog.objects.get(id=1)
>>> e = Entry.objects.get(id=234)
>>> b.entry_set.add(e)            #分配Entry e给Blog b。
```

在上面的例子中，`e.save()`被调用来执行更新。然而，在many-to-many关系中使用`add()`，不会调用任何的`save()`，而是使用`QuerySet.bulk_create()`创建关系。当创建一个关系后，如果你需要执行一些自定义的逻辑，监听`m2m_changed`信号。

  - create(**kwargs)

创建一个新对象，保存它并将它放入到相关对象集合。返回新创建的对象：

```python

>>> b = Blog.objects.get(id=1)
>>> e = b.entry_set.create(headline='Hello',
                           body_text='Hi',
                           pub_date=datetime.date(2005,1,1))
     
#这里不需要调用e.save() - 它已经被保存了。
```

它等同于下面的代码(但是比它更简单):

```python

>>> b = Blog.objects.get(id=1)
>>> e = Entry(
              blog = b,
              headline='Hello',
              body_text='Hi',
              pub_date=datetime.date(2005,1,1))
>>> e.save(force_insert=True)
```

注意，没有必要指定定义关系的model的关键字参数。在上面的例子中，我们不需要传递参数`blog`给`create()`。Django算出新Entry对象的blog字段会被设置为b。

  - remove(obj1[,obj2,...])
    
从相关对象集合中移除指定的model对象：

```python

>>> b = Blog.objects.get(id=1)
>>> e = Entry.object.get(id=234)
>>> b.entry_set.remove(e)        # 从Blog b 中将Entry e解除关联
```

类似于add()，上面的例子中`e.save()`将被调用来执行更新。然而，在一个many-to-many关系中使用`remove()`，将使用`QuerySet.delete()`删除关系，这意味model的`save()`方法不会被调用；如果当一个关系被删除时，你想要执行自定义的代码，监听`m2m_changed`信号。

对于ForeignKey对象，这个方法仅在`null=True`时存在。如果相关的字段不能被设置为`None(NULL)`,那么一个对象不能从一个关系中移除，没有添加到另外一个中。上面的例子中，从`b.entry_set()`中移除`e`等同于`e.blog = None`,因为blog `ForeignKey`没有`null=True`,因此这是无效的

  - clear() 

从相关对象集合中删除所有对象：

```python

>>> b = Blog.objects.get(id=1)
>>> b.entry_set.clear()
```

注意，这不能删除相关对象 - 它只是解关联它们。

像remove()一样，clear()仅在`null=True`的`ForeignKey`上可用。