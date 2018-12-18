### chain

`itertools.chain(*iterables)`
将多个迭代器作为参数,依次返回各个迭代器的内容

```python
def chain(*iterables):
    for it in iterables:
        for element in it:
            yield element
```

#### chain.from_iterable

与 chain 相似，但是需要传入一个嵌套的列表

```python
>>> numbers = list(range(5))
>>> abc = ["a", "b", "c"]
>>> print list(chain.from_iterable([abc, numbers]))
['a', 'b', 'c', 0, 1, 2, 3, 4]
```

### ifilterfalse

`itertools.ifilterfalse(predicate, iterable)`
创建一个迭代器，仅生成 iterable 中 predicate(item)为 False 的项  
如果 predicate 为 None，则返回 iterable 中所有计算为 False 的项  
对函数 func 执行返回假的元素的迭代器

```python
def ifilterfalse(predicate, iterable):

    if predicate is None:
        predicate = bool
    for x in iterable:
        if not predicate(x):
            yield x

>>>  ifilterfalse(lambda x: x%2, range(10))
0 2 4 6 8
```

### imap

`itertools.imap(function, *iterables)`
产生一个通过调用函数生成数据的迭代器，该生成函数的参数逐个来源于各个迭代器参数。
如果迭代器为 None,直接返回参数 tuple

```python
def imap(function, *iterables):
    iterables = map(iter, iterables)
    while True:
        args = [next(it) for it in iterables]
        if function is None:
            yield tuple(args)
        else:
            yield function(*args)
>>> print list(imap1(pow, (2,3,10), (5,2,3)))
[32, 9, 1000]
```

### starmap

`itertools.starmap(function, iterable)`

对序列 iterable 的每个元素作为 function 的参数列表执行, 返回执行结果的迭代器

与 map 相似，但是会将 iterable 参数 item 使用`*`后作为参数传递给 function

```python
def starmap(function, iterable):
    # starmap(pow, [(2,5), (3,2), (10,3)]) --> 32 9 1000
    for args in iterable:
        yield function(*args)
```

### islice

```python
def islice(iterable, *args):

    s = slice(*args)
    it = iter(xrange(s.start or 0, s.stop or sys.maxint, s.step or 1))
    nexti = next(it)
    for i, element in enumerate(iterable):
        if i == nexti:
            yield element
            nexti = next(it)

'''
>>> print list(islice('ABCDEFG', 2))
>>> print list(islice('ABCDEFG', 2, 4))
>>> print list(islice('ABCDEFG', 2, None))
>>> print list(islice('ABCDEFG', 0, None, 2))
'''
```
