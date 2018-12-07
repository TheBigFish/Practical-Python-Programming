## 描述

用于高效循环的迭代函数

这个模块中提供的函数具有和"lazy functional programming language" Haskell 和 SML 相似的特点. 他们都是为了跑得更快和更有效的使用内存. 但他们也被牵扯在一起以表示更为复杂的迭代算法.

## 实现

### starmap

itertools.starmap(function, iterable)

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

## 参考

- [pymotw-itertools](https://pymotw.com/2/itertools/)
- [官方文档](https://docs.python.org/2/library/itertools.html)
