## identity

identity 函数返回传入的单个参数

```python
identity = lambda arg: arg
```

## apply

apply 函数调用传入的第一个参数函数，后续参数做为调用函数的参数传入,args 为 tuple,kwargs 为 dict  
`args or []` 当 args 为 None 时传入空列表

```python
def _apply(f, args=None, kwargs=None):
    return f(*(args or []), **(kwargs or {}))

apply = apply if version_info[0] == 2 else _apply

'''
>>> print apply(operator.add, (2, 3))
5
'''
```

## call

```python
def call(f, *args, **kwargs):
    return f(*args, **kwargs)

'''
>>> print call(operator.mul, (2, 3))
6
```

## flip

flip 返回一个函数，该函数将原来的两个参数对调后调用  
flip(flip(func_a)) is func_a  
使用 setattr 给函数对象增加属性

```python
def flip(f):
    flipper = getattr(f, "__flipback__", None)
    if flipper is not None:
        return flipper

    def _flipper(a, b):
        return f(b, a)

    setattr(_flipper, "__flipback__", f)
    return _flipper
'''
>>> print operator.div(2.0, 5)
>>> print flip(operator.div)(2.0, 5)
>>> print flip(flip(operator.div))(2.0, 5)
0.4
2.5
0.4
'''
```

## Curring

柯里化,将接收多个函数变成接收单个函数的过程

```python
def curry(f, arg, *rest):
    return curry(f(arg), *rest) if rest else f(arg)

'''
>>> def add(first):
>>>     def add(second):
>>>         return first + second
>>>     return add

>>> print(op.curry(add, 2, 3))
5
'''
```

## zipwith

返回函数，对参数先做 zip，在做 starmap
f 调用的参数个数和 zip 之后生成 list 中 tuple 个数要符合

```python
def zipwith(f):
    'zipwith(f)(seq1, seq2, ..) -> [f(seq1[0], seq2[0], ..), f(seq1[1], seq2[1], ..), ...]'
    return F(starmap, f) << zip
'''
>>> zipper = op.zipwith(operator.add)
>>> print(list(zipper([0,1,2], itertools.repeat(10))))
[10, 11, 12]
'''
```
