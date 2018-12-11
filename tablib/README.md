- [简述](#%E7%AE%80%E8%BF%B0)
- [Row](#row)
- [Dataset](#dataset)
  - [property 函数的使用](#property-%E5%87%BD%E6%95%B0%E7%9A%84%E4%BD%BF%E7%94%A8)
  - [`__unicode__` 方法](#unicode-%E6%96%B9%E6%B3%95)
  - [`_register_formats` 方法](#registerformats-%E6%96%B9%E6%B3%95)
  - [`_package`方法](#package%E6%96%B9%E6%B3%95)
- [Databook](#databook)
  - [`detect_format` 方法](#detectformat-%E6%96%B9%E6%B3%95)

## 简述

源码地址：  
[Tablib: format-agnostic tabular dataset library](https://github.com/kennethreitz/tablib.git)  
一个数据集合库
包含了多种数据格式的操作库  
整个源码简洁，直接，清晰

## Row

`class Row(object)`  
数据行类  
一目了然的实现

注意使用 set 检测容器中是否含有某个元素的用法

```python
def has_tag(self, tag):
    """Returns true if current row contains tag."""

    if tag == None:
        return False
    elif isinstance(tag, str):
        return (tag in self.tags)
    else:
        return bool(len(set(tag) & set(self.tags)))
```

## Dataset

`class Dataset(object)`  
Row 的集合
同样简单明了的实现

### property 函数的使用

- 原型
  property(fget=None, fset=None, fdel=None, doc=None)
- Property 对象有三个方法，getter(), setter()和 delete()，用来在对象创建后设置 fget，fset 和 fdel

  ```python
  item = property(fget, fset)

  #等同于

  # make empty property
  item = property()
  # assign fget
  item = item.getter(fget)
  # assign fset
  item = item.setter(fset)
  ```

源码中

```python
def _get_dict(self):
    return self._package()


def _set_dict(self, pickle):
    if not len(pickle):
        return
    ...

dict = property(_get_dict, _set_dict)
```

等同于

```python
dict = property()
# assign fget
dict = dict.getter(_get_dict)
# assign fset
dict = dict.setter(_set_dict)
```

与使用 @property 装饰 相互等价

```python
@property
def dict(self):
    return self._package()
# dict = property(dict, None ...)

@property.setter
def dict(self):
    if not len(pickle):
        return
    ...
# dict = property.setter(dict)
```

### `__unicode__` 方法

漂亮的列表推导，函数式编程的妙用

```python

def __unicode__(self):
    result = []

    # Add unicode representation of headers.
    if self.__headers:
        result.append([unicode(h) for h in self.__headers])

    # Add unicode representation of rows.
    result.extend(list(map(unicode, row)) for row in self._data)

    lens = [list(map(len, row)) for row in result]
    field_lens = list(map(max, zip(*lens)))

    # delimiter between header and data
    if self.__headers:
        result.insert(1, ['-' * length for length in field_lens])


    return '\n'.join(format_string.format(*row) for row in result)
```

### `_register_formats` 方法

动态注册

```python
@classmethod
def _register_formats(cls):
    """Adds format properties."""
    for fmt in formats.available:
        try:
            try:
                # 给类注册 属性 fmt.title
                setattr(cls, fmt.title, property(fmt.export_set, fmt.import_set))
                # 给类注册 导入和导出方法 get_* set_*
                setattr(cls, 'get_%s' % fmt.title, fmt.export_set)
                setattr(cls, 'set_%s' % fmt.title, fmt.import_set)
                cls._formats[fmt.title] = (fmt.export_set, fmt.import_set)
            except AttributeError:
                setattr(cls, fmt.title, property(fmt.export_set))
                setattr(cls, 'get_%s' % fmt.title, fmt.export_set)
                cls._formats[fmt.title] = (fmt.export_set, None)

        except AttributeError:

            cls._formats[fmt.title] = (None, None)
```

`setattr(cls, fmt.title, property(fmt.export_set, fmt.import_set))` 设置 property, 比如 json, 这样可以直接调用
dataset.json, 会触发 `_json.export_set` 的调用

```python
headers = ('first_name', 'last_name', 'gpa')
john = ('John', 'Adams', 90)
george = ('George', 'Washington', 67)
tom = ('Thomas', 'Jefferson', 50)

founders = tablib.Dataset(headers=headers, title='Founders')
founders.append(john)
founders.append(george)
founders.append(tom)

print founders.json
print founders.get_json()


[{"first_name": "John", "last_name": "Adams", "gpa": 90}, {"first_name": "George", "last_name": "Washington", "gpa": 67}, {"first_name": "Thomas", "last_name": "Jefferson", "gpa": 50}]

```

### `_package`方法

- dict 可以作用于 list
  ```python
  ab = [["a", 1], ["b", "2"]]
  cd = dict(ab)
  print cd
  # {'a': 1, 'b': '2'}
  ```
- OrderedDict 的使用

```python
def _package(self, dicts=True, ordered=True):
    """Packages Dataset into lists of dictionaries for transmission."""
    # TODO: Dicts default to false?

    _data = list(self._data)

    if ordered:
        dict_pack = OrderedDict
    else:
        dict_pack = dict

    # Execute formatters
    if self._formatters:
        for row_i, row in enumerate(_data):
            for col, callback in self._formatters:
                try:
                    if col is None:
                        for j, c in enumerate(row):
                            _data[row_i][j] = callback(c)
                    else:
                        _data[row_i][col] = callback(row[col])
                except IndexError:
                    raise InvalidDatasetIndex


    if self.headers:
        if dicts:
            data = [dict_pack(list(zip(self.headers, data_row))) for data_row in _data]
        else:
            data = [list(self.headers)] + list(_data)
    else:
        data = [list(row) for row in _data]

    return data
```

## Databook

`class Databook(object)`  
Dataset 的集合

### `detect_format` 方法

逐个尝试，直到解码正确或者全部失败

```python

# detect in format, such as json
def detect(stream):
    """Returns True if given stream is valid JSON."""
    try:
        json.loads(stream)
        return True
    except ValueError:
        return False


def detect_format(stream):
    """Return format name of given stream."""
    for fmt in formats.available:
        try:
            if fmt.detect(stream):
                return fmt.titless
        except AttributeError:
            pass
```
