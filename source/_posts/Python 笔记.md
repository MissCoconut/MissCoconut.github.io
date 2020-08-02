---
title: Python 基础查缺补漏
date: 2020-08-01 20:18:26
tags: 
- python
---

>本文知识点比较杂且并不深入，多为自己比较模糊的 python 基础特性。
>
>感谢组内@RedFree 大佬的无私分享，以下参考组内分享内容和 b 站视频  [惊呆！Python 竟然还有这样的黑魔法！](https://www.bilibili.com/video/BV1FJ411J7CR) 整理。

# 0x01 基本数据类型

## 推导式

```
[表达式 for 迭代变量 in 可迭代对象 if 条件 / for 循环]
```

- 带判断条件的
```python
In [31]: myList = [i for i in range(1,10) if i > 5]
In [32]: myList
Out[32]: [6, 7, 8, 9]
```
- 折腾字典的
```python
In [33]: myDict = {'key1':'v1','key2':'V2'}
In [37]: ret = {key:value for value,key in myDict.items()}
In [38]: ret
Out[38]: {'v1': 'key1', 'V2': 'key2'}
```
<!-- more -->

- 嵌套循环
```python
In [39]: myList = [str(i) + j for i in range(1,6) for j in 'ABCDE']
In [41]: print(myList)
['1A', '1B', '1C', '1D', '1E', '2A', '2B', '2C', '2D', '2E', '3A', '3B', '3C', '3D', '3E', '4A', '4B', '4C', '4D', '4E', '5A', '5B', '5C', '5D', '5E']
```
同理推导式可应用于字典、集合、元组（注意需要用`tuple()`来定义，直接用`()`会被当成一个生成器）。

```python
In [42]: myList = (i for i in range(1,100))
In [43]: myList
Out[43]: <generator object <genexpr> at 0x04C0D8F0>
In [47]: myList = tuple(i for i in range(1,100))
In [49]: print(myList)
(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99)
```

## 格式化字符串

- `%s` / `%5.3f` C 语言里边那种形式

- `str.format(*args,**kwargs)` 可以指定匿名参数/命名参数

```python
In [62]: print('{1} and {0} and {other}'.format('spam','eggs',other='aaaaa'))
eggs and spam and aaaaa
```
-  `f-string` python 3.6 引入，特别好用
```python
In [63]: fname = 'coconut'
In [64]: lname = 'miss'
In [65]: print(f'Hello,{lname} {fname}')
Hello,miss coconut
```

- f-string 还可以用在类中

```python
class Person:
    def __init__(self,fname,lname):
        self.fname = fname
        self.lname = lname
    def __str__(self):
        return f'hello,{self.fname} {self.lname}'
person = Person('coconut','miss')
print(f'{person}')
```

## collections 加强版的数据类型

- `namedtuple` 带命名的元组
- `deque `双向队列
- `Counter `计数器
- `timeit` 统计函数执行时间，python自带的

- https://docs.python.org/zh-cn/3/library/collections.html
  遇到个啥新的数据类型，不要急于造轮子，先看看 python 标准库里边有现成的没。

# 0x02 函数

## 可变长参数

- `*args` 其他参数，`**kwargs` 关键字参数

```python
def func(*args, **kwargs):
    print(f'args:{args}')
    print(f'kwargs:{kwargs}')
func('a', 'b', others='others')
-----OUTPUT:
args:('a', 'b')
kwargs:{'others': 'others'}
```

## lambda 表达式

只是个表达式，一般适用于很简单的函数逻辑（一个表达式），不用给函数起个名了。

```python
k = lambda x:x+1
print(k(5))
```

## 高阶函数

参数是函数，返回值还是函数
eg. `map()` `filter()`
`functools` 这个包可以参考

## 装饰器`@decorate`

装饰器的参数是被装饰函数。大概理解成 hook 一个函数。

```python
def timethis(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end - start)
        return result
    return wrapper
```

注意第二行的`@wraps`注解，这个注解会保留被包装函数的所有元信息（名字、文档、注释等等)，每次定义装饰器的时候不要落下这个。`@wraps` 把被包装的函数保存在` __wrapped__` 属性中，可以通过这个属性访问原始函数（例如某次调用想去掉装饰器时）。

eg. 

```python
def timethis(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end - start)
        return result
    return wrapper


@timethis
def foo():
    """
    foo function
    :return: None
    """
    while True:
        # time.sleep(5)
        break


if __name__ == "__main__":
    print(foo.__doc__) 
    foo.__wrapped__()  # prints nothing
```



更多装饰器相关，参考:

> https://python3-cookbook.readthedocs.io/zh_CN/latest/chapters/p09_meta_programming.html

## 生成器

- 包含 `yield` 关键字的函数
- **协程**   #todo 挖坑待填
- 下一个内容`next(generator)`
- 输出全部内容 `list(generator)`

# 0x03 面向对象编程

## 魔术方法

- `__call__()`允许类的实例成为可调用的对象
- `__iter__()` 定义被迭代时的行为
- `__getitem__()` 使用`[key]`调用时的行为
- `__str__()` vs. `__repr__()` 
    简单来说 `__str__()` 返回的是给用户看的描述性字符串，`__repr__()`返回的是给其他调用函数使用的字符串.

## 属性描述符 @property

规范属性的赋值取值操作。

```python
class Student(object):
    @property
    def score(self):
        return self._score

    @score.setter
    def score(self, value):
        if value < 0 or value > 100:
            raise ValueError()
        self._score = value


if __name__ == "__main__":
    s = Student()
    s.score = 101  # ValueError
```

把一个 getter 方法变成属性加上`@property`即可，`@property`会创建另一个装饰器 `@score.setter`，负责把一个 setter 方法变成属性赋值，在这个方法中规范属性的操作。

如果某个属性只定义了 getter ，而且下划线开头`(_attr)`，意味着只读属性，直接赋值`(attr=1)`会抛出异常，非要给`_attr`赋值也不是不行，只是不符合规范，建议别这么搞。

属性查找顺序，**类的实例 > 类 > 父类**。

## 抽象基类 Abstract Base Class

> https://python3-cookbook.readthedocs.io/zh_CN/latest/c08/p12_define_interface_or_abstract_base_class.html

父类定义抽象方法 `@abstractmethod`，子类须实现。

## 可变对象 vs 不可变对象

python 变量中保存的是对象的引用，python 中一切传递都是传引用，无论赋值还是函数调用，不存在传值。

- 不可变对象：更改变量所指向的对象值会开辟新的内存空间保存；

- 可变对象：可以直接修改内存中变量所指向的对象的值。

  ![可变对象与不可变对象](https://i.loli.net/2020/08/02/Fvk4UIGZq8lsXaO.png)

- 关于 list

  - `list_a += []`  相当于 `extends()`

  - `list_a = list_a + []` 相当于 new 一个 list 对象并拷贝值

- 关于比较

  ​	`==` 比较内容

  ​	`is` 比较地址

- 深拷贝 （deep copy） vs. 浅拷贝 (shallow copy)

  - 深拷贝：拷贝对象的内容，实现：`copy.deepcopy()`、...
  - 浅拷贝：拷贝引用，实现：`copy.copy()`、`=`、列表切片、...

- 函数默认参数

  - **默认参数一定设置成不可变对象**，此处有坑，印象深刻.

    ```python
    def foo(mylist=[]):
        mylist.append("a")
        print(mylist)
        
    if __name__ == "__main__": 
    	for i in range(5):
            foo()
            
    OUT:
    ['a']
    ['a', 'a']
    ['a', 'a', 'a']
    ['a', 'a', 'a', 'a']
    ['a', 'a', 'a', 'a', 'a']
    ```

    其实这么写 IDE 已经提示了，奈何对于 IDE 的警告就是视而不见...

    ![image-20200802141554993](https://i.loli.net/2020/08/02/r8tUPXevgfIV5Cz.png)

    正确写法：

    ```python
    def foo(mylist=None):
        if mylist is None:
            mylist = []
        mylist.append("a")
        print(mylist)
    ```



# 0x04 程序健壮性

## 运行性能

`dis`模块可以生成一段 python 代码的字节码。

通过字节码，可以排查一段 python 代码到底干了啥，在性能调优上很有帮助。

```python
import dis
def f(*args, **kwargs):
    print(f'args:{args}')
    print(f'kwargs:{kwargs}')
if __name__ == '__main__':
    f('a', 'b', others='others')
    print(dis.dis(f))
```

```python
  4           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('args:')
              4 LOAD_FAST                0 (args)
              6 FORMAT_VALUE             0
              8 BUILD_STRING             2
             10 CALL_FUNCTION            1
             12 POP_TOP
  5          14 LOAD_GLOBAL              0 (print)
             16 LOAD_CONST               2 ('kwargs:')
             18 LOAD_FAST                1 (kwargs)
             20 FORMAT_VALUE             0
             22 BUILD_STRING             2
             24 CALL_FUNCTION            1
             26 POP_TOP
             28 LOAD_CONST               0 (None)
             30 RETURN_VALUE
```

## 上下文管理器

- `with`语句
- 实现上下文管理协议
- 实现了`__enter__()`和`__exit__()`方法

相关参考：
- 实现一个支持`with`语句的对象：
> https://python3-cookbook.readthedocs.io/zh_CN/latest/c08/p03_make_objects_support_context_management_protocol.html?highlight=%E4%B8%8A%E4%B8%8B%E6%96%87

- `@contextmanager` :
> https://python3-cookbook.readthedocs.io/zh_CN/latest/c09/p22_define_context_managers_the_easy_way.html

## 垃圾回收机制

python 中垃圾回收以**引用计数**为主，**标记-清除**、**分代回收**为辅。

- 引用计数：每个对象内部维护一个 counter，在被引用时 +1，解引用 -1，counter =0 时对象被回收。这种策略简单高效，而且具备实时性，但无法解决循环引用的问题，eg.

  ```python
  class PyObject(ctypes.Structure):
      _fields_ = [("refcnt", ctypes.c_long)]
  
  
  def print_memory_info(name):
      pid = os.getpid()
      p = psutil.Process(pid)
  
      info = p.memory_full_info()
      memory = info.uss / (1024 * 1024)
      print(f"{name} used {memory} MB")
  
  
  def foo():
      print_memory_info("foo start")
      _len = 900000
      list_a = [i for i in range(_len)]
      list_b = [i for i in range(_len)]
      list_a.append(list_b)
      list_b.append(list_a)
      addr_a = id(list_a)
      addr_b = id(list_b)
      del list_a, list_b
      print_memory_info("foo end")
      return addr_a, addr_b
  
  
  if __name__ == "__main__":
      addr_a, addr_b = foo()
      # ret = gc.collect(0)
      # print(f"{ret} object(s) collected.")
  
      print(f"list_a ref cnt = {PyObject.from_address(id(addr_a)).refcnt}")
      print(f"list_b ref cnt = {PyObject.from_address(id(addr_b)).refcnt}")
      print_memory_info("main end")
  
  ```

  `foo()`方法返回值为两个 list 的地址，这两个 list 已经用 `del`显式删除，但两个 list 互相持有对方的引用，导致在`foo()`方法结束后并没有回收掉这两个 list ，引用计数还是 1。

  ```python
  foo start used 12.33203125 MB
  foo end used 47.0390625 MB
  list_a ref cnt = 1
  list_b ref cnt = 1
  main end used 47.0390625 MB
  ```

- 标记-清除(mark-sweep)：基于对象可达性分析的垃圾回收算法。

  - 标记阶段：从根节点出发，根节点（简单理解成调用栈或者全局变量）可达的对象标记为可达状态；
  - 回收阶段：没有被标记的对象回收掉。

  这种策略扫描整个堆内存，进行 GC 时需要阻塞正常运行的程序，GC 结束后恢复，性能开销比较大，同时会导致很多小的内存碎片。

- 分代回收

  - 标记清除这种算法，需要扫描整个堆内存，而且被回收掉的对象越少，性能损耗越大；
  - 分代回收大致意思：按照存活时间不同将对象划分成 0、1、2三代，新生对象在 0 代，经过一次 GC 后，如果存活移动到 1 代，2 代为存活时间最长的对象。对于每一代，维护两个变量，counter 和 threshold，counter 表示**当前代分配内存的对象数量 - 上一次回收掉的对象数量**，当 counter 的值超过预定的 threshold 时触发当前代的一次 GC。

- `gc.collect()`

  - 参数可以指定 GC 的代（0 / 1 / 2），不传参数默认进行一次完整的 GC ，返回值为不可达对象数量。


# 0x05 编码风格与设计模式

## 命名约定

1. `__func__`、`__var__` python 保留，别这么写；
2. `__func()`、`__var` 类的 private 方法、属性，无特殊需求不用；
3. `_func()` 、`_var`属于 protected ，被继承或者导入时不会包含；

## 编码规范

https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/python_language_rules/#id15

## 单例模式实现

一个类只有一个实例存在。需要的时候直接用。

```python
def singleton(cls):
    _instance = {}

    @wraps(cls)
    def wrapper(*args, **kwargs):
        if id(cls) not in _instance:
            _instance[id(cls)] = cls(*args, **kwargs)
        return _instance[id(cls)]

    return wrapper


@singleton
class Clz(object):
    def __init__(self):
        pass
```

# 0x06 运行环境

## venv

虚拟环境，各项目运行环境隔离开。
```sh
python -m venv myvenv
# enter venv
soure ./myvenv/bin/active
# exit venv
deactive
```
## pip 加速安装

国内镜像站：豆瓣 / 清华


windows配置：`C:\User\\{User}\pip\pip.ini`

```shell
[global]
index-url='https://pypi.tuna.tsinghua.edu.cn/simple'
```

# 0x07 ipdb 调试器

> 在使用 Django 框架时，遇到一个奇怪的问题... 
>
> 从 PyCharm 启动调试器运行项目，断点打在某个 app 的方法里，执行期间并不会断下来；而断点打在其他文件里，比如 `settings.py`、`urls.py`就可以正常断下。
>
> 这个问题折磨了好久也没整明白是咋回事，算了，不跟他刚，我换个调试器...

## 安装

`pip install ipdb`

## 使用

1. 集成到源码里

   ```python
   import ipdb
   # ...
   ipdb.set_trace() # 添加断点
   ```

   执行到第三行会停下来进入 ipdb 调试。这种方式比较麻烦在，每次添加断点需要修改代码。

2. `python -m ipdb mycode.py`  这样从命令行启动好些

3. 设置断点 `b [文件名:]行号 | 方法`，这样设置的断点在`restart`后依然保留，忽略断点`disable`/ 重新启用`enable`，清除断点 `clear`

4. 下一行 `n`

5. step into `s`

6. 执行到下个断点 `c`

7. 执行到指定行 `j`

8. 执行到返回 `r`

9. 查看源码 `l` ，`ll`输出长一点

   

   以上是平时自己比较常用到的，还有些临时断点`tbreak`、条件断点`condition`之类的不太常用，需要的时候问 Google。

   官方文档 ：https://docs.python.org/3.5/library/pdb.html

# 参考链接

- [Python3 Cookbook](https://python3-cookbook.readthedocs.io/zh_CN/latest)

- [python中的引用传递，可变对象，不可变对象，list注意点](https://www.cnblogs.com/liaohuiqiang/p/9668303.html)

- [第111天：Python 垃圾回收机制](http://www.ityouknow.com/python/2020/01/06/python-gc-111.html)

- [Python垃圾回收](https://dreamgoing.github.io/Python%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6.html)

  