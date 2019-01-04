## count

`itertools.count(start=0, step=1)`
创建一个迭代器，生成从 n 开始的连续整数，如果忽略 n，则从 0 开始计算（注意：此迭代器不支持长整数）  
如果超出了 sys.maxint，计数器将溢出并继续从-sys.maxint-1 开始计算。

```python
def count(start=0, step=1):
    # count(10) --> 10 11 12 13 14 ...
    # count(2.5, 0.5) -> 2.5 3.0 3.5 ...
    n = start
    while True:
        yield n
        n += step

```

等同于`(start + step \* i for i in count())`

## repeat

`itertools.repeat(object[, times])`  
创建一个迭代器，重复生成对象若干次  
times 指定重复计数，如果未提供 times，将无止尽返回该对象。

```python
def repeat(object, times=None):
    if times is None:
        while True:
            yield object
    else:
        for i in xrange(times):
            yield object
'''
>>> repeat(10, 3)
10 10 10
'''
```
