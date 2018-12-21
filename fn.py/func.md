# func.py

> func 模块主要实现两个函数 F 和 curried

## contents

- [func.py](#funcpy)
  - [contents](#contents)
  - [F](#f)
  - [curried](#curried)
  - [版权](#%E7%89%88%E6%9D%83)

## F

核心函数，包装为一个偏函数(partial)，通过操作符 >> << 实现链式操作

```python
class F(object):
    __slots__ = "f",

    def __init__(self, f = identity, *args, **kwargs):
        self.f = partial(f, *args, **kwargs) if any([args, kwargs]) else f

    @classmethod
    def __compose(cls, f, g):
        return cls(lambda *args, **kwargs: f(g(*args, **kwargs)))

    def __ensure_callable(self, f):
        return self.__class__(*f) if isinstance(f, tuple) else f

    def __rshift__(self, g):
        """Overload >> operator for F instances"""
        return self.__class__.__compose(self.__ensure_callable(g), self.f)

    def __lshift__(self, g):
        """Overload << operator for F instances"""
        return self.__class__.__compose(self.f, self.__ensure_callable(g))

    def  __call__(self, *args, **kwargs):
        """Overload apply operator"""
        return self.f(*args, **kwargs)
'''
>>> func = F() << (_ + 10) << (_ + 5)
>>> print(func(10))
25
>>> func = F() >> (filter, _ < 6) >> sum
>>> print(func(range(10)))
15
'''
```

通过 `__compose` 组合调用，将 >> << 操作符左右对象重新生成 F 对象  
`__ensure_callable` 处理 tuple 类型的参数，比如

```python
func = F(operator.add, 3) >> (operator.mul, 5)
print func(2)

# 25
```

## curried

柯里化装饰器  
被装饰函数调用时，先展开 partial 查看已经绑定的参数个数。  
如果和函数初始定义时个数一致，调用函数并返回值；  
如果少于函数定义时的个数，使用 partial 包装后返回 curried。

```python

def curried(func):
    @wraps(func)
    def _curried(*args, **kwargs):
        f = func
        count = 0
        while isinstance(f, partial):
            if f.args:
                count += len(f.args)
            # parial 保存有 func 属性
            f = f.func
        # while 结束，f 变为最原始未被装饰的函数
        # getargspec 获取最初的被装饰函数的参数信息
        spec = getargspec(f)

        if count == len(spec.args) - len(args):
            return func(*args, **kwargs)

        return curried(partial(func, *args, **kwargs))
    return _curried
"""
>>> @curried
... def sum5(a, b, c, d, e):
...     return a + b + c + d + e
...
>>> sum5(1)(2)(3)(4)(5)
15
>>> sum5(1, 2, 3)(4, 5)
15
"""
```

curried 实质是对 partial 嵌套调用的一个包装:

```python
@curried
def sum3(a, b, c):
    return a + b + c
'''
>>> print sum3(1)(2)(3)
>>> print partial(partial(sum3, 1), 2)(3)
6
6
'''
```

## 版权

作者：bigfish  
许可协议：[许可协议 知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)
