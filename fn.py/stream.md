## Stream

### 示例

```python
>>> s = Stream() << range(6) << [6,7]
>>> print s[1]
>>> print list(s)
>>> print list(s[1:2])
1
[0, 1, 2, 3, 4, 5, 6, 7]
[1]
```

`fib = f << [0, 1] << map(add, f, drop(1, f))`  
这里定义了一个惰性求值序列。  
map 函数对序列 f 及 drop(1,f) 序列进行加操作，fib 内部暂存了结果序列，随着内部序列的更新，导致 map 无限增长

```python
from fn import Stream
from fn.iters import take, drop, map
from operator import add

f = Stream()
fib = f << [0, 1] << map(add, f, drop(1, f))

assert list(take(10, fib)) == [0,1,1,2,3,5,8,13,21,34]
assert fib[20] == 6765
assert list(fib[30:35]) == [832040,1346269,2178309,3524578,5702887]
```

### 源码

Stream 实现了一个典型的可迭代类  
\_StreamIterator 实现了一个迭代器，但是它自己没有实现 \_\_iter\_\_ 方法，所以它本身不能迭代  
Stream 实现了 \_\_iter\_\_ 方法，为一个可迭代的类，其返回一个迭代器（\_StreamIterator 实例）

```python
class Stream(object):

    __slots__ = ("_last", "_collection", "_origin")

    class _StreamIterator(object):
        # 任何 对 Stream 的迭代操作都生成新的 _StreamIterator对象
        # 彼此有独立的索引位置，互不干扰
        # 但是共享同一个数据缓存区间 self._stream._collection
        __slots__ = ("_stream", "_position")

        def __init__(self, stream):
            self._stream = stream
            self._position = -1 # not started yet

        def __next__(self):
            # check if elements are available for next position
            # return next element or raise StopIteration
            self._position += 1
            if (len(self._stream._collection) > self._position or
                self._stream._fill_to(self._position)):
                return self._stream._collection[self._position]

            raise StopIteration()

        if version_info[0] == 2:
            next = __next__

    def __init__(self, *origin):
        self._collection = []
        # 初始化时迭代位置设为 -1
        self._last = -1 # not started yet
        self._origin = iter(origin) if origin else []

    def __lshift__(self, rvalue):
        iterator = rvalue() if callable(rvalue) else rvalue
        # 通过 chain 保存迭代器链
        self._origin = chain(self._origin, iterator)
        return self

    def cursor(self):
        """Return position of next evaluated element"""
        return self._last + 1

    def _fill_to(self, index):
        # 索引值小于内部索引，表面已经读出并缓存
        if self._last >= index:
            return True

        # 逐个读出索引
        while self._last < index:
            try:
                n = next(self._origin)
            except StopIteration:
                return False

            # 更新到缓存序列
            self._last += 1
            self._collection.append(n)

        return True

    def __iter__(self):
        return self._StreamIterator(self)

    def __getitem__(self, index):
        if isinstance(index, int):
            # todo: i'm not sure what to do with negative indices
            if index < 0: raise TypeError("Invalid argument type")
            # 将元素从迭代器中读出，直到要获取的索引
            self._fill_to(index)
        elif isinstance(index, slice):
            low, high, step = index.indices(maxint)
            if step == 0: raise ValueError("Step must not be 0")

            # 如果参数是 slice,构造一个 空 Stream 类，并且给其序列增加生成器 map, map调用当前类的 __getitem__ 函数，根据 slice 参数生成对应的序列
            return self.__class__() << map(self.__getitem__, range(low, high, step or 1))
        else:
            raise TypeError("Invalid argument type")

        return self._collection.__getitem__(index)
```
