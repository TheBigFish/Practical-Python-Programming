## 迭代器

可迭代的对象一定不能是自身的迭代器。也就是说，可迭代的对象
必须实现 \_\_iter\_\_ 方法，但不能实现 \_\_next\_\_ 方法。  
另一方面，迭代器应该一直可以迭代。迭代器的 \_\_iter\_\_ 方法应
该返回自身。

## 生成器

- 只要 Python 函数中包含关键字 yield，该函数就是生成器函数。
- 生成器函数会创建一个生成器对象

```python
def gen_123():
    yield 1
    yield 2
    yield 3

g = gen_123()
next(g)
```

gen_123() 返回一个生成器对象，即一个迭代器  
把生成器传给 next(...) 函数时，生成器函数会向前，执行函数定义体中的
下一个 yield 语句，返回产出的值，并在函数定义体的当前位置暂
停。最终，函数的定义体返回时，外层的生成器对象会抛出
StopIteration 异常——这一点与迭代器协议一致。

在 Python 3.3 之前，如果生成器函数中的 return 语句有返回值，那么会报错。现在可以这么
做，不过 return 语句仍会导致 StopIteration 异常抛出。调用方可以从异常对象中获取返
回值。可是，只有把生成器函数当成协程使用时，这么做才有意义。

```python
import re
import reprlib
RE_WORD = re.compile('\w+')
class Sentence:
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)
    def __iter__(self):
        for word in self.words:
            yield word
        return
```

iter() 函数调用 \_\_iter\_\_，返回一个生产器

## 生成器表达式

生成器表达式是语法糖：完全可以替换成生成器函数

```python
>>> def gen_AB():  # ➊
...     print('start')
...     yield 'A'
...     print('continue')
...     yield 'B'
...     print('end.')
...
>>> res1 = [x*3 for x in gen_AB()]  # ➋
start
continue
end.
>>> for i in res1:  # ➌
...     print('-->', i)
...
--> AAA
--> BBB
>>> res2 = (x*3 for x in gen_AB())  # ➍
>>> res2  # ➎
<generator object <genexpr> at 0x10063c240>
>>> for i in res2:  # ➏
...     print('-->', i)
...
start
--> AAA
continue
--> BBBend.
```

❹ 把生成器表达式返回的值赋值给 res2。只需调用 gen_AB() 函数，
虽然调用时会返回一个生成器，但是这里并不使用。  
❺ res2 是一个生成器对象。
