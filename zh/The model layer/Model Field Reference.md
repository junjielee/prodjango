Model field reference
==================

该文档包含Django提供的所有field options和field types的底层细节。

另请参阅：

	 如果内置的字段不能做到这些，你可以尝试[localflavor](https://docs.djangoproject.com/en/1.6/topics/localflavor/),它包含了对特定国家和文化有用的代码套件。你也可以很容易的[编写你自定义的模式字段](https://docs.djangoproject.com/en/1.6/howto/custom-model-fields/)。

注意：

	 技术上讲，这些models被定义在django.db.models.fields,但是为了方便起见，我们导入django.db.models；标准惯例是使用`from django.db import models` 并以`models.<Foo>Field`的形式引用fields。

## Field options

下面的参数对于所有的field types都是可用的，并且他们所有都是可选的。

- Field.null

  如果为`True`,Django将在数据库中以NULL储存空值。默认为`False`。

  在基于字符串的fields例如CharField和TextChar要避免使用`null`，因为空字符串值常被存储为空字符串，而不是NULL。如果一个基于字符串的字段有`null=True`,这意味着它对于“无数据”有两种可能的值：`NULL`,和空字符串。大多数情况，“无数据”有两种可能的值是多余的；Django的约定是使用空字符串，而不是`NULL`。

  对于基于字符串和非基于字符串字段的两种情况，如果你想要表格中允许存在空值，你还需要设置`blank=True`，`null`参数仅影响数据库的储存（查看blank）

  注意：

	 当后台使用Oracle 数据库时，`NULL`值将被储存表示空字符串，不管该属性值是什么。

  如果BooleanField想要接受一个`null`值，使用NullBooleanField代替。

- Field.blank

  如果为`True`,这字段允许为空。默认为False。

  注意，它不同于`null`，`null`是纯数据库相关的，而`blank`是验证相关的。如果一个字段有`blank=True`，那么表单验证允许空值输入。如果字段有`blank=False`，那么该字段是必需的不能为空。

- Field.choices

  一个迭代（例如，一个列表或者元组）包含它自己正好有两项的可迭代对象（例如,[(A,B),(A,B)...]）用作该字段选择。如果它给定，默认的表单控件将是一个带这些选项的选择框，代替标准的文本字段。

  每个元组的第一个元素是被存储的实际值，第二个元素是人类可读的名称。例如：

```python

YEAR_IN_SCHOOL_CHOICES = (
	 	                  ('FR','Freshman'),
		                  ('SO','Sophomore'),
		                  ('JR','Junior'),
		                  ('SR','Senior'),
	                     )
```

  通常，最好在model类内部定义choices，并为每个值定义一个适当名称的常量：

```python
	 
from django.db import models

class Student(models.Model):
	FRESHMAN = 'FR'
	SOPHOMORE = 'SO'
	JUNIOR = 'JR'
	SENIOR = 'SR'
	YEAR_IN_SCHOOL_CHOICES = (
	                          (FRESHMAN, 'Freshman'),
	                          (SOPHOMORE, 'Sophomore'),
	                          (JUNIOR, 'Junior'),
	                          (SENIOR, 'Senior'),
	                         )
year_in_school = models.CharField(max_length=2,
	                              choices=YEAR_IN_SCHOOL_CHOICES,
	                              default=FRESHMAN)

def is_upperclass(self):
	return self.year_in_school in (self.JUNIOR, self.SENIOR)
```

  虽然，你可以在model类的外面定义选择列表并引用它，在model类内部为每个选择定义选项和名称可以用类保存所有的信息，并且使得选项易于引用。（例如，在`Student`模式被导入后，`Student.SOPHOMORE`可以在任何的地方工作。）

  你也可以收集你的可用选项在用于组织目的的命名组中：

```python

MEDIA_CHOICES = (
	                  ('Audio', (
	                             ('vinyl', 'Vinyl'),
	                             ('cd', 'CD'),
	                            )
	                  ),
	                  ('Video', (
	                             ('vhs', 'VHS Tape'),
	                             ('dvd', 'DVD'),
	                            )
	                  ),
	                  ('unknown', 'Unknown'),
	            )  
```

  每个元组的第一个元素是适用于该组的名称。第二个元素是一个2-元组的可迭代对象，每个2-元组包含一个值和一个人类可阅读的名称为选项。分组选项可以与在单一列表中(如这个例子中的unknown选项)的未分组选项组合。

  对于每个含有`choices`集合的模式字段，Django将添加一个方法来为字段的当前值取回人类可阅读的名称。查看数据库API文档中的[get_FOO_display()](https://docs.djangoproject.com/en/1.6/ref/models/instances/#django.db.models.Model.get_FOO_display)。

  最后，注意，choices可以是任何可迭代的对象 - 非必须是列表或者元组。 这可以让你自动构建choices。但是你发现你自己hacking choices为动态的，你可能最好使用一个适当的有外键的数据库表。choices是供没有太大变化的静态数据，如果有的话。

- Field.db_column

  使用数据库列名

- Field.db_index



- Field.db_tablespace
- Field.default
- Field.editable
- Field.error_messages
- Field.help_text
- Field.primary_key
- Field.unique
- Field.unique_for_date
- Field.unique_for_month
- Field.unique_for_year
- Field.verbose_name
- Field.validators

## Field types

- class AutoField(**options)
- class BigIntegerField([**options])
- class BinaryField([**options])
- class BooleanField(**options)
- class CharField(max_length=None[,**options])
- class CommaSeparatedIntegerField(max_length=None[,**options])
- class DateField([auto_now=False,auto_now_add=False,**options])
- class DateTimeField([auto_now=False,auto_now_add=False,**options])
- class DecimalField(max_digits=None,decimal_places=None[,**options])
- class EmailField([max_length=75,**options])
- class FileField(upload_to=None[,max_length=100,**options])

## FileField 和FieldFile

- class FieldFile

  - FieldFile.url
  - FieldFile.open(mode='rb')
  - FieldFile.close()
  - FieldFile.save(name,content,save=True)
  - FieldFile.delete(save=True)

- class FilePathField(path=None[,match=None,recursive=False,max_length=100,**options])

  - FilePathField.path
  - FilePathField.match
  - FilePathField.recursive
  - FilePathField.allow_files
  - FilePathField.allow_folders

- class FloatField(**options)
- class ImageField(upload_to=None[,height_field=None,width_field=None,max_length=100,**option])
- class IntegerField([**options])
- class IPAddressField([**options])
- class GenericIPAddressField([protocol=both,unpack_ipv4=False,**options])
- class NullBooleanField([**options])
- class PositiveIntegerField([**options])
- class PositiveSmallIntegerField([**options])
- class SlugField([max_length=50,**options])
- class SmallIntegerField([**options])
- class TextField([**options])
- class TimeField([auto_now=False,auto_now_add=False,**options])
- class URLField([max_length=200,**options])

## 关系 fields

- class ForeignKey(othermode[,**options])
- class ManyToManyField(othermode[,**opthions])
- class OneToOneField(othermode[,**options])
