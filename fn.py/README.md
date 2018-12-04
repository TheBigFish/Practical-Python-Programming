fn.py 是 python 的函数式编程库(Functional Programming, FP)  
代码仓库：[fn.py](https://github.com/kachayev/fn.py)

## Scala 风格的 Lambda 定义

Python

```python
map(lambda x: x*2, [1,2,3])
```

Scala

```scala
List(1,2,3).map(_*2)
```

Haskell

```haskell
map (2*) [1,2,3]
```

受 Scala 的启发，Fn.py 提供了一个特别的`_`对象以简化 Lambda 语法。

```python
from fn import _

assert (_ + _)(10, 5) = 15
assert list(map(_ * 2, range(5))) == [0,2,4,6,8]
assert list(filter(_ < 10, [9,10,11])) == [9]
```
