- [identity](#identity)
- [apply](#apply)
- [call](#call)
- [flip](#flip)
- [Curring](#curring)
- [zipwith](#zipwith)
- [foldl](#foldl)
- [foldr](#foldr)
- [unfold](#unfold)

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

F 函数的 << 及 >> 都是返回可调用对象  
(F(starmap, f) << zip)(iterable) 相当于  
F(starmap, f)(zip(iterable)) 即  
starmap(f, zip(iterable))

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

## foldl

根据传入的函数参数 f 返回一个折叠函数  
对 it 传入的参数，以从左到右的顺序调用 reduce 折叠  
init 参数作为初始参数传给 reduce

```python
def foldl(f, init=None):
    def fold(it):
        args = [f, it]
        if init is not None: args.append(init)
        return reduce(*args)

    return fold
'''
>>> print foldl(_ + _)([0,1,2,3,4])
10
>>> print foldl(_ * _, 1)([1,2,3])
6
'''
```

## foldr

```python
def foldr(f, init=None):
    def fold(it):
        args = [flip(f), reversed(it)]
        if init is not None: args.append(init)
        return reduce(*args)
'''
>>> print op.foldr(_ / _)([1.0, 2.0, 4.0])
2.0
>>> print foldr(call, 10)([lambda s: s**2, lambda k: k+10])
400
'''
```

参数传递顺序是从右到左，但是，对应的两个参数相对位置不变

```python
>>> print op.foldr(_ / _)([2.0, 2.0, 4.0])
4
```

先计算 2.0/4.0 = 0.5, 然后计算 2.0/0.5 = 4.0  
而不是 4.0/2.0 = 2.0, 然后计算 2.0/2.0 = 1.0

因为 reduce 将每个函数计算的结果作为第一个参数，所以要使用 flip 对调参数。  
从而对于 print foldr(call, 10)([lambda s: s**2, lambda k: k+10]) 能产生直观的结果，而对于 print op.foldr(_ / _)([1.0, 2.0, 4.0]) 产生不直观的结果

## unfold

unfolf 是一个生成器
调用参数 f, f 是一个函数，f 接收一个参数，返回一个 (value, cursor)  
value 在生成器中返回，cursor 当做参数传递给 f 进行下轮调用

```python
def unfold(f):
    def _unfolder(start):
        value, curr = None, start
        while 1:
            step = f(curr)
            if step is None: break
            value, curr = step
            yield value
    return _unfolder
'''
>>> doubler = unfold(lambda x: (x*2, x*2))
>>> list(islice(doubler(10), 0, 10))
[20, 40, 80, 160, 320, 640, 1280, 2560, 5120, 10240]
'''
```
