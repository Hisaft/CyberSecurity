# SPU (安全处理器)
Secretflow框架中的SPU主要起到了一个处理的作用，就像是在姚氏百万富翁问题中提到的，将函数和函数需要的数据传入，在内部进行运算，并将最后处理出来的结果保存（通过reveal的方式查看）。
$$ \text{SPU}(f)(args) \to \text{SPU}(f(args)) $$
以我个人的理解，可以简单实现一下SPU的功能
```python
class spu:
    def __init__(self):
        self.result = None
    def __call__(self, f):
        def inner(*args):
            return f(*args)
        return inner
    def reveal(self):
        return self.result
```
初始化方法建立一个结果的变量。当需要运算的时候，可以将函数和相关参数传入,如果需要结果的话使用reveal方法去查看
```python
def get_winner(a, b) -> bool:
    return a > b

spu_device = spu()
win = spu_device(get_winner)(5,1)
```
上述代码可以粗略的实现这一过程，但问题在于reveal这一部分。当运算完，由于想要达到形似，调用部分选择了柯里化。但柯里化要求返回值，所以在做完运算的时候结果就已经是reveal的状态了。而再次调用reveal，得到的是inner的地址，反而无法正确的得到运算结果。这一点是很大的缺陷。更好的方案还在构思之中