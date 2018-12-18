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
