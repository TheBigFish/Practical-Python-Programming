func 模块主要实现两个函数 F 和 curried

## F

核心函数，包装为一个偏函数(partial)，通过操作符 >> << 实现链式操作

通过 `__compose` 组合调用，

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
