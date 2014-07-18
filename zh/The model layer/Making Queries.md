Making queries
=================

一旦你创建了你的[数据模型](https://docs.djangoproject.com/en/1.6/topics/db/models/),Django自动为你提供一个数据库抽象API，让你创建，检索，更新，删除对象。该文档向你介绍怎么使用该API。参考[数据模型参考](https://docs.djangoproject.com/en/1.6/ref/models/)了解所有不同模型查询选项。

贯穿这个指南（和在参考中），我们将引用下面的模型，它包含一个博客应用：

```python

from django.db import models

class Blog(models.Model):
	name = models.CharField(max_length=100)
	tagline = models.TextField()

	# 在Python 3：def __str__(self):
	def __unicode__(self):
		return self.name

class Author(models.Model):
	name = models.CharField(max_length=50)
	email = models.EmailField()

	# 在Python 3: def __str__(self):
	def __unicode__(self):
		return self.name

class Entry(models.Model):
	blog = models.ForeignKey(Blog)
	headline = models.CharField(max_length=255)
	body_text = models.TextField()
	pub_date = models.DateField()
	mod_date = models.DateField()
	authors = models.ManyToManyField(Author)
	n_comments = models.IntegerField()
	n_pingbacks = models.IntegerField()
	rating = models.IntegerField()

	# 在Python 3：def __str__(self)：
	def __unicode__(self):
		return self.headline
```

## 创建对象

用Python对象表示数据库表数据，Django使用一个直观的系统；一个模型类表示一个数据库表，一个该类的实例表示一个数据库表中的一个特定的记录。

创建一个对象，使用传关键字参数给模型类来实例化它，然后调用`save()`保存它到数据库。

你可以在你期望的任何在Python路径上地方导入模型类。（我们在这里指出这点是因为以前的Django版本要求funky模型导入。）

假设模型在mysite/blog/models.py文件中，有个例子如下：

```python

>>> from blog.models import Blog
>>> b = Blog(name='Beatles Blog',tagline='All the latest Beatles news.')
>>> b.save()
```
它在幕后执行一个`INSERT`SQL语句。Django不会提交到数据库直到你明确的调用`save()`。`save()`方法没有返回值。

另请参阅

`save()`采取了一些这里没有描述的高级选项。查看[save()](https://docs.djangoproject.com/en/1.6/ref/models/instances/#django.db.models.Model.save)的文档了解全部的细节。

在一步中创建和保存一个对象，使用`create()`方法

## 保存更新对象

使用save()，保存更新数据库中已存在对象。

给一个Blog已经存在在数据库中的实例b5，这个例子将改变它的名称和在数据库中更新它的记录：

```python
>>> b5.name = 'New name'
>>> b5.save()
```
它在幕后执行一个`UPDATE`SQL语句。Django不会提交到数据库直到你明确的调用`save()`。

### 保存ForeignKey 和ManyToManyField字段

保存一个ForeignKey字段与保存一个普通字段工作原理完全一样 - 简单分配正确类型的对象给有关的字段。这个例子中更新了一个Entry实例entry的blog属性。假设Entry和Blog的合适的实例已经被保存在了数据库中（因此我们可以在下面取回他们）：

```python

>>> from blog.models import Entry
>>> entry = Entry.objects.get(pk=1)
>>> cheese_blog = Blog.objects.get(name="Cheddar Talk")
>>> entry.blog = cheese_blog
>>> entry.save()
```

更新一个ManyToManyField的工作原理有一点不同 - 在字段上使用`add()`方法来增加一个记录到关系。这个例子是添加Author实例joe到entry对象上。

```python
>>> from blog.models import Author
>>> joe = Author.objects.create(name="Joe")
>>> entry.authors.add(joe)
```

一次增加多条记录到一个ManyToManyField,在调用`add()`时传入多个参数即可，像这样：

```python

>>> john = Author.objects.create(name="John")
>>> paul = Author.objects.create(name="Paul")
>>> george = Author.objects.create(name="George")
>>> ringo = Author.objects.create(name="Ringo")
>>> entry.authors.add(john,paul,george,ringo)
```

如果你试图分配或者添加一个错误类型的对象，Django会报错。

## 检索对象

从数据库中检索对象，是通过一个管理器在你的模型类上构造一个QuerySet。

一个QuerySet代表你的数据库中对象的一个集合。它可以有0个，1个或者多个过滤器。过滤器基于给定的参数缩小了查询结果。用SQL术语，一个QuerySet等于一个`SELECT`语句,一个过滤器是一个限制子句例如`WHERE`,`LIMIT`。

你可以通过使用你模型的管理器来构造一个QuerySet。每个模型至少有一个管理器，默认它被叫做objects。通过模型类直接访问它，像这样：

```python

>>> Blog.objects
<django.db.models.manager.Manager object at ...>
>>> b = Blog(name='Foo',tagline='Bar')
>>> b.objects
Traceback:
    ...
AttributeError: "Manager isn't accessible via Blog instances."
```

** 注意 **

Manager只能通过模型类访问，而不能通过模型实例访问。来强制执行“表级”和“记录级”操作之间的分离。

Manager是模型QuerySet的主要来源。例如，`Blog.objects.all()`返回一个包含数据库中所有Blog对象的QuerySet.

### 检索所有对象

检索对象最简单的方法是从一个表中取回所有的数据。通过在Manager上使用`all()`方法来做到这一点：

```python

>>> all_entries = Entry.objects.all()
```

`all()`方法返回数据库中所有对象的一个QuerySet。

### 使用过滤器检索特定对象

通过`all()`返回的QuerySet描述了数据库表的所有对象。通常情况下，你仅需要从完整对象集合中选择一个子集。

创建这样一个子集，你提取一个初始的QuerySet，增加过滤条件。两种最常用的提取QuerySet的方法是：

- ** filter(\**kwargs) **

返回一个新的QuerySet包含匹配给定查询参数的对象。

- ** exclude(\**kwargs) **

返回一个新的QuerySet包含不匹配给定查询参数的对象。

查询参数（上面函数定义的**kwargs）应该是下面[Field lookups](https://docs.djangoproject.com/en/1.6/topics/db/queries/#field-lookups)中描述的的格式。

例如，获取2006年blog的一个QuerySet,使用`filter()`:

```python

Entry.objects.filter(pub_date__year=2006)
```

使用默认的管理类，它与这个是一样的：

```python

Entry.objects.all().filter(pub_date__year=2006)
```

#### 链接过滤

提取一个QuerySet的结果是一个QuerySet的本身，因此，它可以一起链接提取。例如：

```python

>>> Entry.objects.filter(
                          headline__startswith='What'
                         ).exclude(
                          pub_date__gte=datetime.date.today()
                         ).filter(pub_date__gte=datetime(2005,1,30))
```

它先在数据库中取所有项的初始QuerySet，增加一个过滤器，然后是一个排除，然后是另一个过滤器。最后的结果是一个包含以"What"开始，发布时间在2005年1月20日到当天的所有项的一个QuerySet。

#### 过滤的QuerySets是唯一的

每次你提取一个QuerySet，你会获得一个崭新的绝不会绑定到前面QuerySet的QuerySet。每次提取会创建一个隔离的不同的QuerySet，它可以被存储、使用、复用。

例如：

```python

>>> q1 = Entry.objects.filter(headline__startswith="What")
>>> q2 = q1.exclude(pub_date__gte=datetime.date.today())
>>> q3 = q1.filter(pub_date__gte=datetime.date.today())
```
这三个Queryset是互相隔离的。第一个是基于包含以"What"开始的标题的所有项的QuerySet。第二个是第一个的一个子集，它带一个排除pub_date为今天或者将来的记录的附加条件。第三个也是第一个的子集，它带一个仅选择pub_date是今天或者将来的记录的附加条件。初始的QuerySet（q1）是不受提取过程影响的。

#### QuerySet是懒惰的

QuerySet是懒惰的 - 创建一个QuerySet的动作不会涉及到任何的数据库活动。你可以整天叠加filters，直到QuerySet被计算，否则Django是不会实际运行查询。来看看这个例子：

```python

>>> q = Entry.objects.filter(headline__startswith="What")
>>> q = q.filter(pub_date__lte=datetime.date.today())
>>> q = q.exclude(body_text__icontains="food")
>>> print(q)
```

虽然这个看起来像3个数据库访问，实际上它仅访问数据库一次，是在最后一行(`print(q)`)。一般来说，一个QuerySet的结果是不会从数据库获取的直到你使用他们。当你这样做了，QuerySet被通过访问数据库被计算。了解更多关于计算发生时的确切信息，请查看[When QuerySets are evaluated](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#when-querysets-are-evaluated)。

### 使用get检索单个对象

`filter()`常给你的是一个QuerySet,即使查询匹配到的是单个对象 - 这种情况下，它是一个包含了单个元素的QuerySet。

如果你知道你查询的结果仅有一个对象，那么你可以在管理器上使用`get()`方法,它将直接返回对象：

```python

>>> one_entry = Entry.objects.get(pk=1)
```

你可以用`get()`使用任意的表达式，就像`filter()`一样 - 可以查看下面的[Field lookups](https://docs.djangoproject.com/en/1.6/topics/db/queries/#field-lookups)。

需要注意的是，在使用get()使用filter()的切片[0]时的一个不同点。如果没有查询没有匹配到结果，`get()`将抛出DoesNotExist异常。该异常是查询正被执行的模型类的一个属性 - 因此，上面的代码中，如果没有主键为1的Entry对象，Django将抛出Entry.DoesNotExist。

相似的，如果`get()`查询匹配到超过一个，Django将抛出错误。这种情况下，将抛出MultipleObjectsReturned，它也是该模型类的一个属性。

### 其他QuerySet方法

大多数情况，当你需要从数据查询对象时，你使用`all()`、`filter()`、`exclude()`。然而，这远非所有的。查看[QuerySet API 参考](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#queryset-api)了解各种QuerySet方法的完整列表。

### 限制性的QuerySet

使用Python的数组切片语法的一个子集来限制你的QuerySet为一定数量的结果。这相当于SQL的`LIMIT`和`OFFSET`子句。

例如，返回前5个对象（`LIMIT 5`）:

```python

>>> Entry.objects.all()[:5]
```

返回第6个到第10个对象（`OFFSET 5 LIMIT 5`）:

```python

>>> Entry.objects.all()[5:10]
```

不支持负索引（如，`Entry.objects.all()[-1]`）

通常来说，切片一个QuerySet会返回一个新的QuerySet - 它不会执行查询。一个例外的情况是如果你使用Python切片语法的“step”参数。例如，它实际上会执行查询来返回前10个对象中每2个的列表。

```python

>>> Entry.objects.all()[:10:2]
```

要取回单个对象而不是一个列表（例如，`SELECT foo FROM bar LIMIT 1`），使用单个索引代替切片。例如，下面这个例子返回数据库中Entry按照headline的字母排序后的第一个对象：

```python

>>> Entry.objects.order_by('headline')[0]
```

这大致相当于：

```python

>>> Entry.objects.order_by('headline')[0:1].get()
```
然而，需要注意的是如果没有对象符合给定标准，第一个将抛出IndexError异常，第二个会抛出DoesNotExist。

### 字段查询

字段查询是你怎样制定SQL `WHERE`子句的条件。它们被以关键字参数的形式指定给QuerySet方法`filter()`，`exclude()`和`get()`

基本查询关键字参数是以`field__lookuptype=value`(这里是双下划线)。例如：

```python

>>> Entry.objects.filter(pub_date__lte='2006-01-01')
```

(大致)可翻译为下面的SQL:

```SQL

SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';
```

** 这是如何可能的 **

Python 有能力定义函数接受任意名称-值的参数，这些名称和值在运行时被计算。了解更多信息，请查看Python官方教程的[关键字参数](https://docs.python.org/2/tutorial/controlflow.html#keyword-arguments)

查找中指定的字段必须是模型字段的名称。不过有一个例外，ForeignKey,你可以指定字段名称加上后缀_id。在这种情况下，值参数应该包含外部模型主键的原始值。例如：

```python

>>> Entry.objects.filter(blog_id__exact=4)
```

如果你传递一个无效的关键字参数，查询函数将抛出TypeError。

数据库API大约支持20多个查询类型。在[字段查询参考](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#field-lookups)中有完整的参考。给你试一下都有哪些可用，这里有一些常用的查询供你使用：

 - exact

确切匹配，例如

```python

>>> Entry.objects.get(headline__exact="Man bites dog")
```

将生成SQL代码为：

```SQL

SELECT ... WHERE headline = 'Man bits dog';
```

如果你不提供查找类型 - 就是说，如果你的关键字参数不包含双下划线 - 查找类型会被假定为exact。

例如，下面两个语句是相等的：

```python

>>> Blog.objects.get(id__exact=4)    # 明确的形式
>>> Blog.objects.get(id=4)           # 隐含__exact
```

这是为了便利，因为exact查询是很常用的情况。

 - iexact

不区分大小写匹配。因此，查询：

```python

>>> Blog.objects.get(name__iexact="beatles blog")
```

将匹配标题为"Beatles Blog"的Blog，甚至是"BeAtlES blOG"。

 - contains

区分大小写包含匹配。例如：

```python

Entry.objects.get(headline__contains='Lennon')
```

大致翻译可为这样的SQL：

```SQL

SELECT ... WHERE headline LIKE '%Lennon%';
```

需要注意的是，它匹配这样的标题"Today Lennon honored",而不匹配"today lennon honored"。

它也有不区分大小写的版本，`icontains`。

 - startswith,endswith

starts-with和ends-with查询，相应的。它们也有不区分大小写的版本`istartswith`和`iendswith`。

再一次说明，这仅仅是些皮毛。在[字段查询参考](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#field-lookups)中查看完整的参考。

### 跨关系查询

Django提供了一个功能强大且直观的方式来在查询中追踪关系，在幕后，自动为你添加SQL `JOINS`。跨关系，只是使用模型间相关字段的字段名称，通过双下划线分隔，直到你获得你想要的字段。

这个例子取回博客名称为'Beatles Blog'的所有Entry对象：

```python

>>> Entry.objects.filter(blog__name__exact='Bealtes Blog')
```

这个关系跨度可以达到你想要的深度。

它也可以反向工作。参考"reverse"关系，只需使用模型的小写名称。

这个例子取回所有的至少包含了一个headline中含有"Lennon"的Entry的所有Blog。

```python

>>> Blog.objects.filter(entry__headline__contains='Lennon')
```

如果你正跨多个关系的过滤并且中间的一个模型没有过滤条件的值，Django将会视它为一个空的（所有值为NULL），但是有效的对象。这意味着不会抛出错误。例如，在这个过滤器中：

```python

Blog.objects.filter(entry__authors__name='Lennon')
```

(如果有一个叫Author模型)，如果没有author与一个entry相关关联，它将被视为没有name附加，而不是因为没有author而抛出一个错误。通常这是你期望的发生的结果。这里唯一可能会混淆的是如果你正在使用`isnull`。因此：

```python

Blog.objects.filter(entry__authors__name__isnull=True)
```

将返回auhor上name为空和那些entry上author为空的Blog对象。如果你不想要后面那些对象，你可以写成：

```python

Blog.objects.filter(entry__authors__isnull=False,
                    entry__authors__name__isnull=True)
```

#### 跨多值关系

当你过滤一个基于ManyToManyField的对象或者以反转的Foreignkey时，有两种不同的过滤器你可能会感兴趣。研究Blog/Entry的关系(Blog到Entry是一个one-to-many的关系)。我们可能会想要找出有一个headline中有'Lennon'并且发布于2008年的entry的博客。或者我们想要找出headline中有'Lennon'的entry以及发布于2008年的entry的博客。因为有多个entry关联一个Blog，这两个查询都是可能并且在一些场景下是有意义的。

一个ManyToManyField会产生相同类型的场景。例如，如果一个Entry有一个ManyToManyField叫做tags，我们可能想要找出连接到tags叫做"music"和"bands"的entry,或者我们想要包含一个名称为"music"的tag和状态为"public"的entry。

处理这两种情况，Django有一种处理`filter()`和`exclude()`调用的一致的方法。单个`filter()`调用内部的一切应用于过滤出匹配所有条件的项。后续的`filter()`调用进一步限制对象的集合，但是对于多值的关系，它们适用于链接到主模型的任何对象，这些对象不必被前一个`filter()`调用选择。

这可能会听起来有点困惑，因此需要一个例子来阐明。选择包含有headhline中含有"Lennon"且发布于2008年的entry的博客(同一个entry同时满足两个条件)，我们写成：

```python

Blog.objects.filter(entry__headline__contains='Lennon',
                    entry__pub_date__year=2008)
```

选择headline中有'Lennon'的entry以及发布于2008年的entry的博客，我们写成：

```python

Blog.objects.filter(entry__headline__contains='Lennon').filter(
                    entry__pub_date__year=2008)
```

假设仅有一个博客含有entry包含'Lennon'和entry发布于2008，但是没有发布于2008包含'Lennon'的entry。第一个查询不会返回任何博客，但是第二个查询方法一个博客。

在第二个例子中，第一个过滤器限制QuerySet为所有那些链接到headline中包含'Lennon'的entry。第二个过滤器进一步限制为那些被链接到发布于2008年的entry的博客集合。被第二个过滤器选择的entry可能与或者可能不与第一个过滤器中entry的一样。我们筛选的是每个过滤语句的Blog的项，而不是Entry的项。

所有的这些行为也适用于`exclude()`:在单个`exclude()`语句内的所有条件应用于单个实例(如果这些条件涉及的是相同的多值关系)。在随后的指向相同关系的`filter()`或者`exclude()`调用中的条件最终可能在不同的链接的对象上。

### 过滤器可以引用模型字段

 - class **F**

在到目前为止的例子中，我们通过比较以常数模型字段的值来构造过滤器。但是，如果你想要以相同模型的另一个字段来比较模型字段的值呢？

Django提供`F()`表达式来运行那样的比较。`F()`实例在一个查询中充当引用一个模型字段的角色。在查询过滤器中这些引用可以用来比较相同模型实例不同字段值。

例如，查找所有博客中评论数大于pingback的entry的列表，我们构造一个`F()`来引用pingback计数，然后在查询中使用那个`F()`对象：

```python

>>> from django.db.models import F
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))
```

Django支持`F()`对象与常数和其他`F()`对象的加，减，乘，除和取模运算使用。查找所有博客中评论数大于2倍的pingback数的entry，我们修改这个查询条件：

```python

>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks') * 2)
```

查询所有rating小于pingback数和评论数之和的entry，我们修改查询：

```python

>>> Entry.objects.filter(rating__lt=F('n_comments') + F('n_pingbacks'))
```

我们也可以在`F()`对象使用双下划线符合来跨关系。带双下划线的F()对象将引入任何联接需要访问相关对象。例如，取回作者名称与博客名称一样的entry，我们修改查询：

```python

>>> Entry.objects.filter(authors__name=F('blog__name'))
```

对于date/datetime字段，你可以添加或者减轻一个timedelta对象。下面这个例子将返回修改时间超过发布时间3天的entry：

```python

>>> from datetime import timedelta
>>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))
```

Django 1.5中新增：

 .bitand()和.bitor()

`F()`对象现在支持按位运算操作，`.bitand()`和`.bitor()`，例如：

```python

>>> F('somefield').bitand(16)
```

Django 1.5中的改变：

以前的操作符`&`和`|`不再产生按位运算操作，使用`.bitand()`和`.bitor()`代替。

### pk查询快捷方式

为了便利，Django提供pk查询快捷方式，它代表"主键"。

在例子Blog模型中，主键是id字段，因此这三个语句是相等的：

```python

>>> Blog.objects.get(id__exact=14) # 明确形式
>>> Blog.objects.get(id=14)        # __exact被隐藏
>>> Blog.objects.get(pk=14)        # pk意味着id__exact
```

pk的使用没有被限制为`__exact`查询 - 任何查询术语都可以与pk联合来在模型主键上执行查询：

```python

# 获取id为1，4，7的博客
>>> Blog.objects.filter(pk__in=[1,4,7])

# 获取id大于14的所有博客
>>> Blog.objects.filter(pk__gt=14)
```

pk查询也可以交叉连接。例如，这三个语句是相等的：

```python

>>> Entry.objects.filter(blog__id__exact=3) # 明确形式
>>> Entry.objects.filter(blog_id=3)         # __exact被隐藏
>>> Entry.objects.filter(blog__pk=3)        # __pk意味着__id__exact
```

### LIKE语句中的转义百分号和下划线

等同于SQL语句中`LIKE`的字段查询(iexact,contains,icontains,startswith,istartswith,endswith,iendswith)将自动转义两个用于`LIKE`语句的特殊字符 - 百分号和下划线。(在一个`LIKE`语句中，百分号表示一个多字符通配符，下划线表示单个字符通配符。)

这意味着事情应该工作直观，因此抽象不漏(This means things should work intuitively, so the abstraction doesn’t leak.)。例如，查询包含百分号的entry，我们仅需要像使用其他字符串一样使用百分号：

```python

>>> Entry.objects.filter(headline__contains='%')
```

Django会为你转义，生成的SQL像这样：

```SQL

SELECT ... WHERE headline LIKE '%\%%';
```

下划线也是一样的，Django会为你透明的处理下划线和百分号。

### 缓存和QuerySet

每个的Queryset都包含一个缓存来减少数据库的访问。理解它是怎么样工作的可以让你写出最高效的代码。

在一个新建的QuerySet中，缓存是空的。第一次QuerySet被评估 - 故，数据库发生查询 - Django保存查询结果在QuerySet的缓存中并且返回被明确请求的结果（例如，下一个元素，如果QuerySet被遍历）。QuerySet后续的评估将复用缓存的结果。

在心里保持这个缓存行为，因为如果你没有正确使用你的QuerySet它可能会咬你。例如，下面将创建两个QuerySet，评估他们，然后扔掉：

```python

>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])
```

这意味着相同的数据库查询将被执行两次，有效的两次数据库加载。还有一种可能性，两个列表可能不包括相同的数据库记录，因为一个Entry在两次请求之间可能被添加或者被删除。

为了避免这个问题，简单的保存QuerySet和重用他：

```python

>>> queryset = Entry.objects.all()
>>> print(p.headline for p in queryset) # 计算查询集合
>>> print(p.pub_date for p in queryset) # 重用缓存
```

#### 何时QuerySet不被缓存

Queryset并不是常常都缓存他们的结果。当仅评估查询部分的QuerySet时，检查缓存，但是如果它不被评估，那么被后续查询返回的项不会被缓存。特别的是，这意味着使用数组的切片或者索引将不会填充缓存。

例如，在查询对象中重复获取某个索引，将每次都会检查数据库：

```python

>>> queryset = Entry.objects.all()
>>> print queryset[5]  # 查询数据库
>>> print queryset[5]  # 再次查询数据库
```

然而，如果整个QuerySet已经被评估了，则用检查缓存来替代：

```python

>>> queryset = Entry.objects.all()
>>> [entry for entry in queryset]   # 查询数据库
>>> print queryset[5]               # 使用缓存
>>> print queryset[5]               # 使用缓存
```

这里还有一些其他行为的例子，整个queryset的结果被评估，因此填充缓存：

```python

>>> [entry for entry in queryset]
>>> bool(queryset)
>>> entry in queryset
>>> list(queryset)
```

** 注意 **

简单地打印QuerySet不会填充缓存，这是因为它仅仅是调用`__repr__()`返回整个queryset的一个切片。

## 使用Q对象进行复杂查询

 - class **Q**

关键词参数查询 - 在`filter()`中，等等 - 是`AND`。如果你需要执行更多的复杂查询(例如，用`OR`语句查询)，你可以使用Q对象：

Q对象(django.db.models.Q)是一个用于封装关键词参数集的对象。这些关键词参数是上面"字段查询"中指定的。

例如，这个Q对象封装单个`LIKE`查询:

```Python

from django.db.models import Q
Q(question__startswith='What')
```

Q对象可以使用`&`和`|`操作符连接。当一个操作符用于两个Q对象时，它将产生一个新的Q对象。

例如，这个语句将产生单个Q对象来表示两个"question__startswith"查询的`OR`:

```python

Q(question__startswith='Who') | Q(question__startswith='What')
```

这相当于下面的SQL `WHERE`子句：

```SQL

WHERE question LIKE 'Who%' OR question LIKE 'What%'
```

你可以通过使用`&`和`|`操作符连接Q对象，并使用括号分组来编写任意复杂度的查询语句。Q对象也可以使用`~`操作符来取反，允许联合正常的查询和否定(NOT)查询:

```python

Q(question__startswith='Who') | ~Q(pub_date__year=2005)
```

每个采用关键词参数的查询函数(例如，`filter()`,`exclude()`,`get()`)也可以传递一个或者多个Q对象作为位置参数（没有命名的）。如果你传递多个Q对象给一个查询函数，那么该参数将被执行`AND`操作，例如：

```python

Poll.objects.get(
                 Q(question__startswith='Who'),
                 Q(pub_date=date(2005,5,2)) | Q(pub_date=date(2005,5,6))
                )
```

大致可翻译为如下的SQL：

```SQL

SELECT * FROM polls 
WHERE question LIKE 'Who%' 
AND (pud_date = '2005-05-02' OR pub_date = '2005-05-06');
```

查询函数混合使用Q对象和关键词参数。提供给查询函数的参数(Q对象或者关键词参数)都是执行`AND`操作。然而，如果提供了一个Q对象，那么它必须在任何关键词参数之前。例如：

```python

Poll.objects.get(
                 Q(pub_date=date(2005,5,2)) | Q(pub_date=date(2005,5,6)),
                 question__startswith='Who')
                )
```

这是有效的查询，相当于前一个；但是：

```python

# 无效的查询
Poll.objects.get(
                 question__startswith='Who',
                 Q(pub_date=date(2005,5,2)) | Q(pub_date=date(2005,5,6))
                )
```

这是无效的查询。

** 另请参阅 **

Django 单元测试中的[OR 查询示例](https://github.com/django/django/blob/master/tests/or_lookups/tests.py)展示了一些Q的可能用法。

## 比较对象

比较两个模型实例，使用标准的Python比较操作符，双等于符号:==。幕后，它会比较两个模型主键的值。

使用上面的Entry例子，下面两个语句是相等的：

```python

>>> some_entry == other_entry
>>> some_entry.id == other_entry.id
```
如果模型的主键不叫id,没关系，比较常使用主键，不管它叫什么。例如，如果一个模型的主键叫name，下面两个语句是相等的：

```python

>>> some_obj == other_obj
>>> some_obj.name == other_obj.name
```

## 删除对象

删除方法名称是`delete()`。这个方法立即删除对象并且没有返回值。

例如：

```python

e.delete()
```

你也可以批量的删除对象。每个QuerySet都有一个`delete()`方法,它会删除该QuerySet的所有成员。

例如，下面这个例子会删除pub_date为2005年的所有Entry对象。

```python

Entry.objects.filter(pub_date__year=2005).delete()
```

请记住，只要有可能，这将以纯SQL被执行，因此单个的对象实例的`delete()`方法在过程中未必会被调用。如果你在模型类上提供了一个自定义的`delete()`方法并且确切的想要它被调用，那么你需要手动的删除那个模型的实例(例如，遍历QuerySet并且在每个对象上单独调用`delete()`)，而不是在QuerySet上使用批量的`delete()`方法。

当Django删除一个对象时，默认地，它模拟SQL约束条件`ON DELETE CASCADE`的行为 - 换句话说，任何有外键指向要被删除的对象将会与它一起被删除。例如：

```python

b = Blog.objects.get(pk=1)
# 将删除这个Blog和所有它的Entry对象。
b.delete()
```

这个级联行为是可通过传递on_delete参数给ForeignKey来自定义的。

需要注意的是`delete()`方法仅仅是QuerySet的方法，它未暴露于manager本身。这是一种安全机制，以防止你意外的请求`Entry.objects.delete()`,而删除所有的entry。如果你想要删除所有的对象，那么你必须明确的请求一个完整的QuerySet。

```python

Entry.objects.all().delete()
```

## 复制模型实例

虽然没有内置方法来复制模型实例，它可以很简单的创建一个所有字段值都是复制的新实例。这种情况下，你仅需要设置pk为None。用我们的博客为例：

```python

blog = Blog(name='My blog',tagline='Blogging is easy')
blog.save() # blog.pk == 1

blog.pk = None
blog.save() # blog.pk == 2
```

如果你使用继承，事情就会变得复杂得多，定义一个Blog的子类：

```python

class ThemeBlog(Blog):
	theme = models.CharField(max_length=200)

django_blog = ThemeBlog(name='Django',tagline='Django is easy',theme='python')
django_blog.save() # django_blog.pk == 3
```

由于继承的工作原理，你必须同时设置pk和id为None：

```python

django_blog.pk = None
django_blog.id = None
django_blog.save() # django_blog.pk == 4
```

这个过程不会复制关系对象。如果你想复制关系，你必须写一点代码。在我们的例子中，Entry有一个多对多的字段指向Author:

```python

entry = Entry.objects.all()[0]     # 以前的一些entry
old_authors = entry.authors.all()  
entry.pk = None
entry.save()
entry.authors = old.authors        # 保存新的多对多的关系    
```

## 一次更新多个对象

有时你想要为QuerySet中所有对象的一个字段设置一个特定的值。你可以使用`update()`方法来做到这个。例如：

```python

# 更新发布于2007年所有的entry的headline
Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')
```

`update()`方法会被立即应用并且返回查询匹配到的行数(这可能不等于更新的行数，因为有些行可能已经有了新值)。被更新的QuerySet的唯一限制是它仅能访问一个数据库表，模型主要的表。你可以基于关系字段过滤，但是你仅能更新模型主要表的列。例如：

```python

>>> b = Blog.objects.get(pk=1)

# 更新属于这个博客的所有headline
>>> Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')
```

请注意，`update()`方法直接被转换为一个SQL语句。这是一个直接更新的批量操作。它不会在你的模型上运行任何的`save()`方法，或发送pre_save或者post_save信号(这是调用`save()`的结果)，或者遵循auto_now字段选项。如果你想要保存QuerySet中的每项并确保在每个实例上`save()`方法被调用,你不需要任何特别的函数来处理它，仅需要循环遍历他们然后调用`save()`:

```python

for item in my_queryset:
	item.save()
```

调用更新函数也可以使用`F()`对象来更新一个字段基于模型中另一个字段的值。这对于基于当前值更新计算器是非常有用的。例如，为博客中每个entry增加pingback的计数：

```python

>>> Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)
```

然而，不像`F()`在filter和exclude，当你在更新中使用`F()`时，你不能使用联接 - 你仅能引用被更新本地模型的字段。如果你视图在`F()`对象中使用联接，会抛出FieldError：

```python

# 这将抛出一个FieldError
>>> Entry.objects.update(headline=F('blog__name'))
```

## 关系对象

当你在模型中定义了一个关系(例如，ForeignKeyField,OneToOneField或者ManyToManyField)，那个模型的实例将有一个便利的API来访问相关的对象。

使用本页顶部的模型，例如，一个Entry对象e可以通过访问blog的属性：e.blog来获得它关联的Blog对象。

(幕后，这个功能是通过Python [descriptors](http://users.rcn.com/python/download/Descriptor.htm)来实现的。这个对你并不重要，我们在这里指出来仅仅是为了好奇)

Django也为关系的"另一边"创建了便利的API访问器 - 从相关模型连接到定义关系的模型。例如，一个Blog对象b可以通过entry_set属性`b.entry_set.all()`来访问所有相关的Entry对象的列表。

本章中使用的例子是定义在本页的顶部Blog、Author、Entry模型。

### 一对多关系

#### 前向

如果一个模型定义了一个ForeignKey，那么该模型的实例可以通过该模型简单的属性来访问相关的（foreign）对象。

例如：

```python

>>> e = Entry.objects.get(id=2)
>>> e.blog   # 返回相关的Blog对象
```

你可以通过一个外键属性来获取和设置。如你所期望的，改变外键不会保存到数据库直到你调用`save()`。例如：

```python

>>> e = Entry.objects.get(id=2)
>>> e.blog = some_blog
>>> e.save()
```

如果一个外键字段有`null=True`设置(即,允许NULL值)，你可以分配None来移除关系。例如：

```python

>>> e = Entry.objects.get(id=2)
>>> e.blog = None
>>> e.save()  # "UPDATE blog_entry SET blog_id = NULL ...;"
```

向前访问一对多的关系会在第一次相关对象被访问的时候缓存，后续访问相同对象实例的外键是缓存的。例如：

```python

>>> e = Entry.objects.get(id=2)
>>> print(e.blog) # 访问数据接收相关的Blog
>>> print(e.blog) # 不访问数据；使用缓存的版本
```

需要注意的是，使用[select_related()](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#django.db.models.query.QuerySet.select_related) QuerySet方法递归要提前预填充缓存。例如：

```python

>>> e = Entry.objects.select_related().get(id=2)
>>> print(e.blog) # 不访问数据库；使用缓存版本
>>> print(e.blog) # 不访问数据库；使用缓存版本
```

#### 后向

如果一模型有一个外键，外键模型的实例可以访问Manager来返回第一个模型的所有的实例。默认的，这个管理器的名称是FOO_set，其中FOO是原模型的名称，小写。这个管理器返回QuerySet，可以被上面"检索对象"章节描述的过滤和处理。

例如：

```python

>>> b = Blog.objects.get(id=1)
>>> b.entry_set.all() # 返回Blog的所有Entry的对象

# b.entry_set 是一个返回QuerySet的管理器
>>> b.entry_set.filter(headline__contains='Lennon')
>>> b.entry_set.count()
```

你可以在ForeignKey定义中设置related_name参数来复写FOO_set的名称。例如，如果Entry模型被修改为`blog = ForeignKey(Blog,related_name='entries')`,那么上面的代码看起来会是这样的：

```python

>>> b = Blog.objects.get(id=1)
>>> b.entries.all() # 返回Blog的所有Entry的对象

# b.entry_set 是一个返回QuerySet的管理器
>>> b.entries.filter(headline__contains='Lennon')
>>> b.entries.count()
```

除了定义在"检索对象"里定义的QuerySet方法，ForeignKey Manager有额外的方法用于处理相关对象集合。每个的方法的简介如下，完整的细节请查看[related objects reference](https://docs.djangoproject.com/en/1.6/ref/models/relations/)

 - **add**(obj1,obj2, ...)

添加指定的模型对象到相关的对象集

 - **create**(**kwargs)

创建一个新对象，保存它并将它放到关系对象集里。返回新建的对象。

 - **remove**(obj1,obj2, ...)

从相关对象集里移除指定的对象。

 - **clear**()

从相关对象集里移除所有的对象。

为关系集合一次分配多个对象，仅需要从任意的可迭代对象给它分配。该可迭代对象包含对象实例，或者是主键值的列表。例如：

```python

b = Blog.objects.get(id=1)
b.entry_set = [e1,e2]
```
在这个例子中，e1和e2是Entry的实例，或者整数的主键值。

如果`clear()`方法可用,在迭代对象(这个例子中，是列表）被添加到集合之前任意的预先存在的对象将会被从entry_set中移除。如果`clear()`方法不可用，迭代对象中任何对象都会被添加而不会移除任意已存在的元素。

本章描述的每个"逆向"操作会立即影响数据库。每次添加，创建和删除会立即自动的保存到数据库。

### 多对多关系

一个多对多关系的两端会自动获得API访问另一端。这个API的工作原理像上面讲的一对多关系上的"后向"。

唯一的不同是属性的命名：定义了ManyToManyField的模型使用使用该字段本身的属性名，"逆向"模型使用原模型的小写模型名称，加上'_set'(这跟一对多关系一样)。

下面这个例子可以让你更好的理解：

```python

e = Entry.objects.get(id=3)
e.authors.all()  # 返回这个Entry的所有作者对象
e.authors.count()
e.authors.filter(name__contains='John')

a = Author.objects.get(id=5)
a.entry_set.all() # 返回这个作者的所有Entry对象
```

像ForeignKey,ManyToManyField也能指定related_name。上面的例子中，如果Entry中ManyToManyField指定了`related_name='entries'`，那么每个Author实例都会有一个entries属性来代替entry_set。

### 一对一关系

一对一关系与多对一关系非常相似。如果你在模型中定义了一个OneToOneField，那么那个模型的实例可以通过模型的属性来访问关系对象。

例如：

```python

class EntryDetail(models.Model):
	entry = models.OneToOneField(Entry)
	details = models.TextField()

ed = EntryDetail.objects.get(id=2)
ed.entry # 返回相关的Entry对象
```

不同点在"逆向"查询上。一对一关系的关系模型也可以访问Manager对象，但是这个Manager表示单个对象，而不是对象的集合。

```python

e = Entry.objects.get(id=2)
e.entrydetail # 返回相关的EntryDetail对象
```

如果没有对象被分配给这个关系，Django将会抛出DoesNotExist异常。

你可以为逆向关系分配实例，就像你分配给前向关系一样。

```python

e.entrydetail = ed
```

### 后向关系是怎么做到的？

其他对象关系映射要求你要在两边都定义关系。Django的开发者认为这违反了DRY原则(不要你重复)，因此Django仅需要你在一端定义关系。

但是这是怎么做到的呢，给定一个模型类不知道哪个其他模型类与它相关直到那些其他模型类被加载？

答案就在INSTALLED_APPS设置中，第一次任何模型都会被加载，Django会遍历INSTALLED_APPS中的每个模型并且根据需要在内存中创建后向关系。本质上，INSTALLED_APPS的一个功能就是告诉Django整个model的域。



### 通过相关对象查询

涉及相关对象的查询遵循涉及普通字段值查询的相同规则。当为一个查询指定值做匹配时，你可以使用对象实例的本身，或者对象的主键值。

例如，如果你有id=5的Blog对象b，下面的3个查询是相同的：

```python

Entry.objects.filter(blog=b)    # 使用对象实例查询
Entry.objects.filter(blog=b.id) # 使用实例的id查询
Entry.objects.filter(blog=5)    # 直接使用id查询
```

## 编写纯SQL查询

如果你发现你自己需要写一个SQL查询，使用Django的数据库映射器来处理太复杂了，你可以手工编写SQL。Django有一组用于写纯SQL的选项。查看[执行纯SQL查询](https://docs.djangoproject.com/en/1.6/topics/db/sql/)。

最后，重要的是要注意，Django数据库层仅仅是你数据库的一个接口。你可以通过其他工具，程序语言或者数据库框架来访问数据。数据库没有Django特征。