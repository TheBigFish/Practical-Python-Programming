## for 迭代

```python
>>> s = 'ABC'
>>> for char in s:
...     print(char)
...
A
B
C
```

等同于

```python
>>> s = 'ABC'
>>> it = iter(s)
>>> while True:
...     try:
...         print(next(it))
...     except StopIteration:
...         del it
...         break
...
A
B
C
```
