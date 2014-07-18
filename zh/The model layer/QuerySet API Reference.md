QuerySet API Reference
=========================

这个文档描述了QuerySet API的细节。它建立在已呈现的 [model](https://docs.djangoproject.com/en/1.6/topics/db/models/) 和 [database query guide](https://docs.djangoproject.com/en/1.6/topics/db/queries/) 的材料基础之上，因此你可能需要在阅读该文档之前先要阅读和理解上面提到的文档。

整个这个参考，我们将使用 [database query guide](https://docs.djangoproject.com/en/1.6/topics/db/queries/) 中的 [example Weblog models](https://docs.djangoproject.com/en/1.6/topics/db/queries/#queryset-model-example) 。

## QuerySet何时被评估

在内部, QuerySet 被构造，过滤，切片，和普通的传递并没有实际的访问数据库。实际没有发生数据库活动直到你做一些什么来评估QuerySet。

你可以通过下面的方法评估一个 QuerySet：

- Iteration

QuerySet是一个可迭代对象，你第一次遍历它的时候，它会执行它的数据库查询。例如，这会打印数据库中所有entry的headline。

```python

for e in Entry.objects.all():
	print(e.headline)
```

注意，如果你想要做的是确定是否至少有一个结果存在，你不需要这么做。可以使用`exist()`更有效。

- Slicing

如 [Limiting QuerySet](https://docs.djangoproject.com/en/1.6/topics/db/queries/#limiting-querysets) 中的解释，一个QuerySet可以用Python的组织切片语法进行切片。切片一个未评估的QuerySet常返回另一个未评估的QuerySet，但是如果你在切片语法中使用"step"参数，Django将会执行数据库查询，并返回一个列表。切片一个已被评估(部分的或者全部的)的QuerySet也会返回一个列表。

- Picking/Caching

查看下面的章节了解当 [pickling QuerySet](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#pickling-querysets) 时涉及到的细节。本条最重要的目的是它的结果是从数据库中读取的。

- repr()

当你对QuerySet调用`repr()`时，QuerySet会被评估。这是为了方便在Python交互式解释器使用，因此当你交互式使用API的时候，可以立即看到你结果。

- len()

当你对QuerySet调用`len()`时，QuerySet会被评估。正如你期望的，它将返回结果列表的长度。

注意，如果你想要确定集合中记录数，不需要使用`len()`。在数据库级处理计数会更有效，使用SQL `SELECT COUNT(*)`，正是这个原因，Django提供了一个`count()`方法。查看下面的`count()`方法。

- list()

通过对QuerySet调用`list()`可以强制评估它。例如：

```python

entry_list = list(Entry.objects.all())
```

警告，这可能会有很大的内存开销，因为Django将在内存中加载列表的每个元素。与此相反，遍历一个QuerySet会根据你的需要利用你的数据库加载数据和实例化对象。

- bool()

在布尔上下文中测试一个QuerySet，例如，使用`bool()`,`or`,`and`或者`if`语句，将会导致查询被执行。如果至少有一个结果QuerySet为True，否则为False。例如：

```python

if Entry.objects.filter(headline="Test"):
	print("There is at least one Entry with the headline Test")
```

注意，如果你想要做的是确定是否至少有一个结果存在，你不需要这么做。可以使用`exist()`更有效（参见下面的介绍）。

### Pickling QuerySets

If you pickle a QuerySet, this will force all the results to be loaded into memory prior to pickling. Pickling is usually used as a precursor to caching and when the cached queryset is reloaded, you want the results to already be present and ready for use (reading from the database can take some time, defeating the purpose of caching). This means that when you unpickle a QuerySet, it contains the results at the moment it was pickled, rather than the results that are currently in the database.

If you only want to pickle the necessary information to recreate the QuerySet from the database at a later time, pickle the query attribute of the QuerySet. You can then recreate the original QuerySet (without any results loaded) using some code like this:

```python

>>> import pickle
>>> query = pickle.loads(s)       # 假设 's' 为pickled的字符串
>>> qs = MyModels.objects.all()
>>> qs.query = query              # 欢迎原 'query'
```

The query attribute is an opaque object. It represents the internals of the query construction and is not part of the public API. However, it is safe (and fully supported) to pickle and unpickle the attribute’s contents as described here.

** You can’t share pickles between versions **

Pickles of QuerySets are only valid for the version of Django that was used to generate them. If you generate a pickle using Django version N, there is no guarantee that pickle will be readable with Django version N+1. Pickles should not be used as part of a long-term archival strategy.

## QuerySet API

虽然你不会常常手动创建一个 QuerySet - 你一般会通过一个Manager - 这里有一个QuerySet的正式声明：

- class **QuerySet**(*[model=None,query=None,using=none]*)

通常，当你与QuerySet交互时，你会通过链接过滤器来使用它。为了使它工作，多数的QuerySet方法返回新的QuerySet。这些方法将会在本章的后面有详细介绍。

QuerySet 类有两个公共属性，你可以用于自检：

1. ordered
如果 QuerySet 被排序则为 True  — 即，有一个 `order_by()` 子句或者模型中有一个默认的排序。否则为 False 。
  
2. db
如果这个查询现在被执行，将会使用的数据库。

** Note **

The query parameter to QuerySet exists so that specialized query subclasses such as GeoQuerySet can reconstruct internal query state. The value of the parameter is an opaque representation of that query state and is not part of a public API. To put it simply: if you need to ask, you don’t need to use it.

### 返回新 QuerySet 的方法

Django 提供了一系列精致的方法来修改QuerySet返回结果的类型或者执行SQL查询的方法。

#### filter

- **filter**(**kwargs)

返回一个新 QuerySet，它包含了匹配到给定查询参数的对象。

这个查询参数(**kwargs)在下面的 [Field lookups](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#id4) 中有描述。多参数会通过 SQL 语句中相关的 AND 连接。 

#### exclude

- **exclude**(**kwargs)

返回一个新 QuerySet，它包含了不匹配给定参数的对象。

这个查询参数(**kwargs)在下面的 [Field lookups](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#id4) 中有描述。多参数会通过 SQL 语句中相关的 AND 连接，而整个被封闭在`NOT()`中。

这个例子不包括所有pub_date大于2005-1-3 `AND` headline是"Hello"的entry:

```python

Entry.objects.exclude(pub_date__gt=datetime.date(2005,1,3),headline='Hello')
```

用SQL表示，相当于：

```SQL

SELECT ...
WHERE NOT (pub_date > '2005-1-3' AND headline = 'Hello')
```

这个例子不包括所有pub_date大于2005-1-3 `OR` headline是"Hello"的entry:

```python

Entry.objects.exclude(pub_date__gt=datetime.date(2005,1,3)).exclude(headline='Hello')
```

用SQL表示，相当于：

```SQL

SELECT ...
WHERE NOT pub_date > '2005-1-3'
AND NOT headline = 'Hello'
```

需要注意的是，第二个例子有更多的限制。

#### annotate

- **annotate**(\*args,**kwargs)

#### order_by
#### reverse
#### distinct
#### values
#### values_list
#### dates
#### datetimes
#### none
#### all
#### select_related
#### prefetch_related
#### extra
#### defer
#### only
#### using
#### select_for_update

### 不返回 QuerySet 的方法

#### get
#### create
#### get_or_create
#### bulk_create
#### count
#### in_bulk
#### iterator
#### latest
#### earliest
#### first
#### last
#### aggregate
#### exists
#### update
#### delete

### Field lookups

#### exact
#### iexact
#### contains
#### icontains
#### in
#### gt
#### gte
#### lt
#### lte
#### startswith
#### istartswith
#### endswith
#### iendswith
#### range
#### year
#### month
#### day
#### week_day
#### hour
#### mintue
#### second
#### isnull
#### search
#### regex
#### iregex

### Aggregation functions

#### Avg
#### Count
#### Max
#### Min
#### StdDev
#### Sum
#### Variance