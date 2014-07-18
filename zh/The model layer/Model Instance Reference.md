Model instance reference
===========================

这个文档描述了Model API的细节。它建立在[model](https://docs.djangoproject.com/en/1.6/topics/db/models/)和[数据查询](https://docs.djangoproject.com/en/1.6/topics/db/queries/)向导提供的材料上，因此你可能要在阅读本文之前先阅读和理解这些文档。

整个这个参考，我们将使用到[数据库查询向导](https://docs.djangoproject.com/en/1.6/topics/db/queries/)中提到的[Weblog models例子](https://docs.djangoproject.com/en/1.6/topics/db/queries/#queryset-model-example)

## 创建对象

创建一个model的新实例，仅是实例化它，像其他任何的Python类一样：

- class Model(**kwargs)

关键字参数只是你在model中定义字段的名称。需要注意的是，以任何方式触及到数据库实例化一个model；你都需要`save()`。

注意：

     你可以通过重写__init__方法来尝试自定义model。如果你这么做，注意不要改变调用签名，任何的改变可能会阻碍正在保存的model实例。相比重写\__init\__,你可以尝试以下方法之一：

  1. 在model类中添加一个类方法：

```python

from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=100)

    @classmethod
    def create(cls,title):
        book = cls(title=title)
        # do something with the book
        return book

book = Book.Create("Pride and Prejudice")
```
  
  2. 在一个自定义管理器上添加一个方法（常用）

```python

class BookManager(models.Model):
    def create_book(self,title):
        book = self.create(title=title)
        # do something with the book
        return book

class Book(models.Model):
    title = models.CharField(max_length=100)
    objects = BookManager()

book = Book.objects.create_book("Pride and Prejudice")
```

## 验证对象

验证一个model有三个步骤：

1. 验证model的字段 - [Model.clean_fields()](https://docs.djangoproject.com/en/1.6/ref/models/instances/#django.db.models.Model.clean_fields)
2. 验证model整体   - [Model.clean()](https://docs.djangoproject.com/en/1.6/ref/models/instances/#django.db.models.Model.clean)
3. 验证字段的唯一性- [Model.validate_unique()](https://docs.djangoproject.com/en/1.6/ref/models/instances/#django.db.models.Model.validate_unique)

当你调用一个model的`full_clean()`方法时，所有的3个步骤都会执行。

当你使用一个[ModelForm](https://docs.djangoproject.com/en/1.6/topics/forms/modelforms/#django.forms.ModelForm)时，调用`is_valid()`将为包含在表单里所有的字段执行这些验证步骤。查看[ModelForm 文档](https://docs.djangoproject.com/en/1.6/ref/models/instances/)了解更多信息。如果你计划自己处理验证错误，或者你已经从需要验证的ModelForm中排除字段，你仅需要调用一个model的`full_clean()`方法。

  - Model.**full_clean**(exclude=None,validate_unique=True)

Django 1.6中的变化：validate_unique参数被添加用于允许跳过`Model.validate_unique()`。早先，`Model.validate_unique()`常被`full_clean`调用。

这个方法调用`Model.clean_fields()`,`Model.clean()`,和`Model.validate_unique()`(如果`validate_unique=True`,按照这个顺序抛出一个ValidationError异常，它有一个message_dict属性包含所有3个步骤的错误。)

可选参数exclude用于提供一个字段名列表，该列表中的字段将会从验证和清理中排除。ModelForm使用该参数来排除不存在在你的表单上被验证的字段，因为抛出的任何错误都不能由用户自行解决。

需要注意的是，当你调用你的model的save()方法的时候，`full_clean()`不会被自动调用。当你想要为你自己手动创建的models运行一步model验证时，你需要手动的调用它。例如：

```python

from django.core.exception import ValidationError

try:
    article.full_clean()
exception ValidationError as e:
    # Do something based on the errors contained in e.message_dict.
    # Display them to a user, or handle them programatically.
    pass
```

第一步`full_clean()`为各个字段执行清理。

  - Model.**clean_fields**(exclude=None)

这个方法将验证你model上的所有字段。可选exclude参数让你提供一个要从验证中去除的字段列表。如果任何一个字段验证失败将抛出一个ValidationError。

第二步`full_clean()`执行是调用`Model.clean()`,该方法可以被重写，来在你的model上执行自定义的验证。

  - Model.**clean**()

该方法用于提供自定义的模式验证，如果有需要并可以在你的模式上修改属性。例如，你能用它为一个字段自动提供一个值，或者验证需要访问一个以上的单个字段。

```python
     
import datetime
from django.core.exceptions import ValidationError
from django.db import models

class Article(models.Model):
    ...
    def clean(self):
        # Don't allow draft entries to have a pub_date.
        if self.status == 'draft' and self.pub_date is not None:
            raise ValidationError('Draft entries may not have a publication date.')
        if self.status == 'published' and self.pub_date is None:
            self.pub_date = datetime.date.today()
```

然而，需要注意的是，像`Model.full_clean()`一样,当你调用模式的`save()`方法的时候，`clean()`方法不会被调用。

任何被`Model.clean()`抛出的ValidationError异常将被储存在一个特定的键错误字典键中，`NON_FIELD_ERRORS`，它用于被连接到整个模式，而不是一个特定的字段的错误。

```python

from django.core.exceptions import ValidationError,NON_FIELD_ERRORS

try:
    article.full_clean()
except ValidationError as e:
    non_field_errors = e.message_dict[NON_FIELD_ERRORS]
```

最后，`full_clean()`会检查你的模式中的任意唯一性约束。

  - Model.**validate_unique**(exclude=None)

这个方法类似于`clean_fields()`，但是它是验证模式上的所有唯一性约束而不是单个字段的值。可选exclude参数让你提供一个要从验证中去除的字段列表。如果任意字段验证失败，将抛出一个ValidationError异常。

需要注意的是，如果你提供了一个exclude参数给`validate_unique()`，任何unique_together约束涉及到你提供的字段都不会被检查。

## 保存对象

保存一个对象到数据库,调用`save()`：

  - Model.**save**([force_insert=False,force_update=False,using=DEFAULT_DB_ALIAS,update_fields=None])

如果你想要自定义保存行为，你可以复写这个`save()`方法。见[重写预定义模式方法](https://docs.djangoproject.com/en/1.6/topics/db/models/#overriding-model-methods)了解详情。

模式保存过程含有一些细节；请参阅下面的章节。

### 自增主键 

如果一个模式有AutoField - 一个自增主键 - 那么在你的对象第一次调用`save()`时，一个自动增加的值将被计算和作为属性保存：

```python

>>> b2 = Blog(name='Cheddar Talk',tagline='Thoughts on cheese.')
>>> b2.id      # 返回为None，因为b还没有ID
>>> b2.save()  
>>> b2.id      # 返回你新对象的ID
```

在你调用`save()`之前没有办法知道ID的值，因为该值是被数据库计算的而不是Django。

为了方便，每个模式默认有一个名称为id的AutoField,除非你明确的在模式的一个字段上指定`primary_key=True`。查看[AutoField](https://docs.djangoproject.com/en/1.6/ref/models/fields/#django.db.models.AutoField)文档了解更多细节。

** pk 属性 **

  - Model.**pk**

不管是你自己定义一个主键字段，还是Django为你提供一个，每个模式都还有一个叫pk的属性。它的行为就像模式上的一个正常属性，但实际上它是一个别名，其属性是模式主键字段。你可以读取和设置它的值，就像你操作其他属性一样，它将在模式中更新当前字段。

** 明确指定自增主键的值 **

如果一个含有AutoField字段的模式，但是当你保存的时候，想要明确定义一个新对象的ID，只是在保存之前明确定义他，而不依赖自动生成的ID:

```python

>>> b3 = Blog(id=3,name='Cheddar Talk',tagline='Thoughts on chees.')
>>> b3.id   # 返回为3
>>> b3.save()
>>> b3.id   # 返回为3
```
    
如果你手动分配自动主键值，需确保不使用一个已经存在的主键值。如果你创建一个带明确主键值而该主键已在数据库中存在的新对象，Django将假设你是改变已存在的记录而不是新建一个。

以上面的'Cheddar Talk'的blog为例，这个例子将覆盖在数据库上一条记录：

```python

>>> b4 = Blog(id=3,name='Not Cheddar',tagline='Anything but cheese.')
>>> b4.save()  # 覆盖前面ID=3的记录！
```

查看[Django 怎么知道是更新还是插入？](https://docs.djangoproject.com/en/1.6/ref/models/instances/#how-django-knows-to-update-vs-insert),下面，是这种情况发生额原因。

明确指定自增主键值常用于批量保存的对象，当你确信不会有主键冲突的时候。

### 当你保存的时候发生了什么 

当我们保存一个对象时，Django执行以下步骤：

1. ** 发送预存储(pre-save)的信号 **。发送`django.db.models.signals.pre_save`信号，允许任何方式监听该信号来做一些自定义动作。
2. ** 预处理数据 **。对象的每个字段被要求执行该字段可能需要执行的任何自动数据修改。   
   大多数的字段不需要预处理 - 字段数据保持原样。预处理仅用于有特定行为的字段上。例如，如果你的模式有带`auto_now=True`的DateField字段，预存储阶段将改变对象中的数据，确保该时期字段包含当前的日期戳。（我们文档还没有包含所有有“特定行为”字段的列表。）   
3. ** 准备数据 **。每个字段被要求在能被写入到数据库中的数据类型中提供当前的值。
   大多数字段不需要数据准备。简单的数据类型，如整数和字符串，是以一个Python对象“准备写”的。然而，更复杂的数据类型往往需要进行一些修改。
   例如，DateField字段使用Python datetime对象存储数据。数据库不存储datetime对象，因此字段的值必须转换为符合ISO标准的时间字符串插入到数据库中。   
4. ** 插入数据到数据库中 **。预处理，准备数据，然后组合成用于插入到数据库中的SQL语句。
5. ** 发送保存后(post-save)信号 **。发送`django.db.models.signals.post_save`信号，允许任何方式监听该信号来做一些自定义动作。

### Django 怎么知道是更新还是插入？ 

你可能注意到了Django数据库对象使用相同的`save()`方法来创建和改变对象。Django需要使用`INSERT`或者`UPDATE`SQL语句。特别的是，当你调用`save()`时，Django遵循以下的算法：

- 如果对象的主键属性被设置了一个值，计算结果为真（即，一个不是None或者空字符串的值），Django执行一个`UPDATE`操作
- 如果对象的主键属性没有设置或者如果`UPDATE`没有更新任何东西，Django将执行一个`INSERT`操作。

这里有个疑难杂症是当你保存一个新对象的时候，应该小心，不要明确指定一个主键值，如果你不能保证主键的值未被使用。欲了解更多关于这个细微的差别，请上面的[明确指定自增主键的值](https://docs.djangoproject.com/en/1.6/ref/models/instances/#explicitly-specifying-auto-primary-key-values)和下面的[强制插入或者更新](https://docs.djangoproject.com/en/1.6/ref/models/instances/#forcing-an-insert-or-update)

Django 1.6 中的改变：

当主键值设置的时候，之前的Django是做一个`SELECT`。如果`SELECT`发现一行，那么Django做`UPDATE`操作否则做一个`INSERT`。旧算法会导致在UPDATE里有一个或多个查询。在一些罕见的情况下，数据库不会报告一行被更新了，即使数据库包含对象主键值的行。一个例子是PostgreSQL的`ON UPDATE`触发器返回为NULL。在这种情况下，可以通过设置`select_on_save`选项为True来恢复到旧算法。

** 强制插入或者更新 **

在一些罕见的情况下，需要强制`save()`方法来执行一个SQL `INSERT`，不可回落执行一个`UPDATE`。反之亦然，如果可能的话，执行更新操作，而不是插入一个新行。在这些情况下，你是传递`force_insert=True`或者`force_update=True`参数给`save()`方法。明显的，传递两个参数是错误的：你不能同时执行插入和更新。

你需要使用这些参数的情况是非常罕见的。Django几乎都能做正确的事情，尝试覆盖将导致难以跟踪的错误。此功能仅适用于高级用途。

使用`update_fields`将强制更新类似于`force_update`。

### 在已存在的字段上更新属性 

有时你需要在一个字段上执行一个简单的算术运算任务，例如增加或者减小当前的值。要实现这个，显而易见的方法是像这样做：

```python

>>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
>>> product.number_sold += 1
>>> product.save()
```

如果从数据取回的旧的number_sold是10，那么11将被写回到数据库中。

此序列在包含有竞争条件时，有一个标准更新问题。如果执行另一个线程已保存一个更新后的值后，当前线程取回的是旧值，当前线程仅保存旧值加1，而不是新（当前）值加1。

该过程可通过表达相对于原字段值的更新值来使得更健壮和稍快，而不是显式的赋一个新值。Django提供[F()表达式](https://docs.djangoproject.com/en/1.6/topics/db/queries/#query-expressions)执行这类的相对更新。使用F()表达式，前面的日子被表达为：

```python

>>> from django.db.models import F
>>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
>>> product.number_sold = F('number_sold') + 1
>>> product.save()
```

该方法不从数据库使用初始值。 相反，它使数据库做基于`save()`被执行时当前值的更新。

一旦对象被保存，你必须重新加载该对象，以便访问被应用到更新字段的实际值：

```python

>>> product = Product.objects.get(pk=product.pk)
>>> print(product.number_sold)
42
```
了解更多细节，请查看文档[F()表达式](https://docs.djangoproject.com/en/1.6/topics/db/queries/#query-expressions)和他们的[在更新查询中的使用](https://docs.djangoproject.com/en/1.6/topics/db/queries/#topics-db-queries-update)。

### 指定哪些字段保存 

Django 1.5 中新增

如果`save()`在关键字参数update_field里传递一个字段名的列表，那么仅是在列表里面的字段会更新。如果你只想更新对象里一个或者几个字段，这是可取的。防止所有模式字段更新会有一定的性能提升。例如：

```python

product.name = 'Name Changed again'
product.save(update_field=['name'])
```

update_fields参数可以是任何包含字符串的可迭代对象。一个空的update_fields可迭代对象将跳过保存。None值将更新所有字段。

指定update_fields将强制更新。

当保存一个通过延迟模式加载（`only()`或者`defer()`）获得的模式时，仅从数据库加载的字段被更新。事实上，在这种情况下有一个自动的update_fields。如果你分配或者改变任何延迟字段值，该字段将被添加到update fields里。

## 删除对象

  - Model.**delete**([using=DEFAULT_DB_ALIAS])

为对象发一个SQL `DELETE`。这仅仅在数据库中删除对象；Python实例将依然存在并且在它的字段上还有数据。

了解更多细节，包括怎么批量删除对象，查看[删除对象](https://docs.djangoproject.com/en/1.6/topics/db/queries/#topics-db-queries-delete)

如果你想自定义删除行为，你可以重写`delete()`方法。查看[重写预定义的模型方法](https://docs.djangoproject.com/en/1.6/topics/db/models/#overriding-model-methods)了解更多细节。

## 其他模式实例方法 

有几个对象方法有特殊的用途。

注意：在Python 3,由于所有字符串原本被认为是Unicode，仅使用`__str__()`方法(`__unicode__()`方法废弃了)。如果你想兼容Python2，你可以用`python_2_unicode_compatible()`修饰你的模式类。

### \_\_unicode\_\_ 

  - Model.\_\_**unicode**\_\_()

无论何时对对象调用`unicode()`时，`__unicode__()`方法将被调用。Django在一些地方使用`unicode(obj)`（或者相关函数，str(obj)）。值得注意的是，在Django管理工具显示一个对象并且当它显示一个对象时作为一个值插入到模板中。因此，你应该总是从`__unicode__()`方法中显示模式的很好的，人类可读的表现形式。

例如：

```python

from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)

    def __unicode__(self):
        return u'%s %s'%(self.first_name,self.last_name)
```

如果在你的模式中定义一个`__unicode__()`方法而不是`__str__()`方法,Django将自动为你提供一个`__str__()`方法，它调用`__unicode__()`,然后将结果正确地转换为UTF-8编码的字符串对象。这里有一个推荐的开发实践：仅定义`__unicode__`，当需要的时候，让Django转换为字符串对象。

### \_\_str\_\_ 

  - Model.\_\_**str**\_\_()

无论何时对对象调用`str()`时，`__str__()`方法将被调用。该方法在Django内部的直接主要用途是模式的`repr()`的输出在任意地方显示（例如，在调试中显示）。因此，你应该为对象的`__str__()`返回一个不错的，人类可读的字符串。如果你有合理的`__unicode__()`方法了，不需要在没处都放`__str__()`方法。

前面的`__unicode__()`例子可用`__str__()`写成这样：

```python

from django.db import models
from django.utils.encoding import force_bytes

class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)

    def __str__(self):
        # 注意，这里使用Django.utils.encoding.force_bytes()
        # 是因为first_name,last_name将是unicode字符串。
        return force_bytes('%s %s' %(self.first_name,self.last_name))
```

### get_absolute_url 

  - Model.**get_absolute_url**()

定义一个`get_absolute_url()`方法告诉Django怎样为一个对象计算规范的URL。对调用者，该方法应显示返回一个可用于通过HTTP指向该对象的字符串。

例如：

```python

def get_absolute_url(self):
    return "/people/%i/" % self.id
```

(虽然这个代码是正确和简单的，它可能不是写这种方法最可移植的方式。`reverse()`函数通常是最好的办法。)

例如：

```python

def get_absolute_url(self):
    from django.core.urlresolvers import reverse
    return reverse('people.views.details',args=[str(self.id)])
```

Django使用`get_absolute_url()`的一个地方是在admin app中。如果一个对象定义了该方法，对象的编辑页面将有一个"View on site"的链接直接跳转到对象的公共视图，通过`get_absolute_url()`给定的。

相似的，a couple of other bits of Django，例如 [syndication feed framework](https://docs.djangoproject.com/en/1.6/ref/contrib/syndication/)，当它被定义时使用`get_absolute_url()`。如果要得你的模式实例每个有一个唯一的URL，你应该定义`get_absolute_url()`。

在模板中使用`get_absolute_url()`是比较好的做法，而不是硬编码你的对象URL。例如，这个模板代码不是很好：

```html

<!-- 不好的模板代码。应避免使用! -->
<a href="/people/{{ object.id }}/">{{ object.name }}</a>
```

下面这模板代码会好一些：

```html

<a href="{{ object.get_absolute_url }}">{{ object.name }}</a>
```

这里的逻辑是，如果你改变对象的URL结构，即使是简单的事情，如拼写错误，你不得不去检查创建URL的每个地方。Specify it once, in get_absolute_url() and have all your other code call that one place.

** 注意 **

从`get_absolute_url()`中返回的字符串**必须**只能包含ASCII字符（URI规范RFC 2396 要求）并且如果有需要,需要URL编码。

代码和模板调用`get_absolute_url()`应该可以直接使用结果而没有任何的进一步的处理。如果你使用的unicode字符串包含了ASCII以外的字符，你可能希望使用`django.utils.encoding.iri_to_uri()`函数来帮助处理这个问题。

** The permalink decorator **

** 警告 **
permalink decorator不再推荐使用，你应该在你的`get_absolute_url()`方法体中使用`reverse()`方法来代替。

在Django的早期版本中，在`get_absolute_url()`内部没有一个简单的方法使用URLconf文件中定义的URLs。这意味着你需要在URLconf和`get_absolute_url()`两个地方定义URL。permalink decorator被添加来解决这个DRY原则的违规。然而，自从介绍了`reverse()`,我们没理由再使用`permalink`。

 - **permalink**()

该修饰器需要一个URL模式的名称（视图名称或者URL模式名称）和一个位置或者关键字参数列表并使用URLconf模式构造正确的，完整的URL。它返回一个正确URL的字符串，并且在正确的位置上所有的参数会被替换。

固定链接修饰器一个Python级的相当于url模板标签和一个`reverse()`函数的高级封装。

一个例子可以介绍这么使用`permalink()`,建议你的URLconf 包含下面的一行：

```python

(r'^people/(\d+)/$','people.views.details')
```

你的模式应该有一个`get_absolute_url()`,像如下一样：

```python

from django.db import models

@models.permalink
def get_absolute_url(self):
    return ('people.views.details',[str(self.id)])
```

类似的，如果你有一个URLconf实体，像这样的：

```python

(r'/archive/(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/$',archive_view)
```

你可以使用`permalink()`如下：

```python

@models.permalink
def get_absolute_url(self):
    return ('archive_view',(),{
            'year':self.created.year,
            'month':self.created.strftime('%m'),
            'day':self.created.strftime('%d')
    } )
```

请注意，在这种情况下我们为第二个参数指定了一个空序列，因为我们仅想要传递关键字参数，没有位置参数。

通过这种方法，你关联了模式的绝对路径和用于显示它的视图，任何地方都没有重复的视图URL信息。与以前一样，你仍可以在模板中使用`get_absolute_url()`方法。

在某些情况下，如使用通用视图或者多模式自定义视图的复用，指定视图的方式可能会混淆反向URL的匹配(因为多模式指向相同的视图)。对于这种情况，Django命名URL模式。使用一个命名的URL模式，它可以为一个模式提供一个名称，然后应用该名称而不是视图函数。命名的URL模式是通过调用url函数替代模式元组来定义的。

```python

from django.conf.urls import url
url(r'^people/(\d+)/$','blog_views.generic_detail',name='people_view')
```

然后使用那个名称执行URL的反向解析而不是view名称。

```python

from django.db import models

@models.permalink
def get_absolute_url(self):
    return ('people_view',[str(self.id)])
```

了解更多关于命名的URL参数，查看[URL dispatch documentation](https://docs.djangoproject.com/en/1.6/topics/http/urls/)

## 额外的实例方法 

除`save()`,`delete()`外，模式对象还有如下的方法：

  - Model.**get_FOO_display**()

每个含有choices集合的字段，对象有一个`get_FOO_display()`方法，FOO是字段的名称。该方法返回字段人类可读的值。

例如：

```python

from django.db import models

class Person(models.Model):
    SHIRT_SIZES = (
                  (u'S',u'Small'),
                  (u'M',u'Medium'),
                  (u'L',u'Large'),
    )
    name = models.CharField(max_length=60)
    shirt_size = models.CharField(max_length=2,choices=SHIRT_SIZES)
```

```python

>>> p = Person(name="Fred Flintstone",shirt_size ="L")
>>> p.save()
>>> p.shirt_size
u'L'
>>> p.get_shirt_size_display()
u'Large'
```

  - Model.**get_next_by_FOO**(\**kwargs)
  - Model.**get_previous_by_FOO**(\**kwargs)

对于每个不含`null=True`的DateField和DateTimeField，对象有`get_next_by_FOO()`和`get_previous_by_FOO()`方法，FOO为字段的名称。它返回相对于日期字段的下一个和前一个对象。在适当的时候，抛出一个DoesNotExist异常。

这两种方法都将使用该模型的默认管理器中执行自己的查询。如果你需要使用自定义的管理器模拟过滤，或者想要执行一次性自定义过滤，两个方法都接受可选的关键字参数，它[Field lookups](https://docs.djangoproject.com/en/1.6/ref/models/querysets/#field-lookups)有描述。

需要注意的是，在完全相同的日期值的情况下，这两个方法将使用主键作为仲裁。这保证了不会有记录被跳过或者重复。这也意味着你不能使用这些方法在未保存的对象上。  