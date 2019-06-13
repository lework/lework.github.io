---
layout: post
title:  "Python 小手册 - 异常处理"
categories: python
tags: python
excerpt: Python 2.7的异常处理
auth: lework
---
* content
{:toc}
## Python 小手册 - 异常处理

> 提示：本文档以`python2.7`版本为例


异常处理无外乎几件事: 断言(`assert`)和抛错(`raise`), 检查(`try`), 捕获(`except`), 处理(`except`,`else`,`finally`)。

异常即是一个事件，该事件会在程序执行过程中发生，影响了程序的正常执行。一般情况下，在Python无法正常处理程序时就会发生一个异常。当Python脚本发生异常时我们需要捕获处理它，否则程序会终止执行。异常作为事件，不需要在程序里传送结果标志或显式地测试它们。

异常是Python对象，表示一个错误。异常可以作为类被定义, 也可以人为引发异常。

异常可以作为控制流, 通过异常情况或人为引发异常, 可以执行代码流控制, 实现比较高级的”goto”效果. 例如在for循环内引发错误,可以跳到外面几层的某个`try..except`内。


`try` 语句

```python

try:
	<statement>        # 检查语句
except NameError:
	<statement>        # 捕获'NameError'异常
except (Error1, Error2, Error3):
	<statement>        # 捕获'Error1, Error2, Error3'多个异常
except KeyError,e:
    <statement>        # 将异常传给变量e,收集作为数据进行处理
except IndexError as e:
    <statement>        # 将异常传给变量e,收集作为数据进行处理
except:                 
	<statement>        # 捕获所有的异常
else:
	<statement>        # 如果没有异常发生，则执行
finally:
	<statement>        # 不管有无异常都会执行

```

`raise` 语句

```python
raise <name>    # 手工地引发异常
raise <name>, <data>    # 传递一个附加的数据(一个值或者一个元组),要是不指定参数,则为None.
raise Exception(data)    # 和上面等效.
raise [Exception [, args [, traceback]]]  # 第三个参数是用于跟踪异常对象,基本不用.

try:
	if (i>10):
		raise TypeError(i)
	elif (i<0):
		raise ValueError,i

# 下面的e实际是返回错误的对象实例.
except TypeError,e:
	print str(e)+" for i is larger than 10!"
except ValueError,e:
	print str(e)+" for i is less than 0!"
else:
	print "i is between 0 and 10~"
```


`assert` 语句 

```python
try:
	assert 1 == 0,'one does not equal zero'
except AssertionError,args:
	print '%s:%s' % (args.__class__.__name__,args)

---
AssertionError:one does not equal zero
```

异常


自定义异常，最好直接或间接继承自异常的类

```python
class Networkerror(RuntimeError):
   def __init__(self, arg):
      self.argsm = arg
try:
   raise Networkerror("Bad hostname")
except Networkerror,e:
   print e.argsm
```


### 常见异常
---

|异常名称|描述|
|--------|----|
|BaseException|所有异常的基类|									
|SystemExit|解释器请求退出|
|KeyboardInterrupt|用户中断执行(通常是输入^C)|
|Exception|常规错误的基类|
|StopIteration|迭代器没有更多的值|
|GeneratorExit|生成器(generator)发生异常来通知退出|
|StandardError|所有的内建标准异常的基类|
|ArithmeticError|所有数值计算错误的基类|
|FloatingPointError|浮点计算错误|
|OverflowError|数值运算超出最大限制|
|ZeroDivisionError|除(或取模)零(所有数据类型)|
|AssertionError|断言语句失败|
|AttributeError|对象没有这个属性|
|EOFError|没有内建输入,到达EOF标记|
|EnvironmentError|操作系统错误的基类|
|IOError|输入/输出操作失败|
|OSError|操作系统错误|
|WindowsError|系统调用失败|
|ImportError|导入模块/对象失败|
|LookupError|无效数据查询的基类|
|IndexError|序列中没有此索引(index)|
|KeyError|映射中没有这个键|
|MemoryError|内存溢出错误(对于Python解释器不是致命的)|
|NameError|未声明/初始化对象(没有属性)|
|UnboundLocalError|访问未初始化的本地变量|
|ReferenceError|弱引用(Weakreference)试图访问已经垃圾回收了的对象|
|RuntimeError|一般的运行时错误|
|NotImplementedError|尚未实现的方法|
|SyntaxError|Python语法错误|
|IndentationError|缩进错误|
|TabError|Tab和空格混用|
|SystemError|一般的解释器系统错误|
|TypeError|对类型无效的操作|
|ValueError|传入无效的参数|
|UnicodeError|Unicode相关的错误|
|UnicodeDecodeError|Unicode解码时的错误|
|UnicodeEncodeError|Unicode编码时错误|
|UnicodeTranslateError|Unicode转换时错误|
|Warning|警告的基类|
|DeprecationWarning|关于被弃用的特征的警告|
|FutureWarning|关于构造将来语义会有改变的警告|
|OverflowWarning|旧的关于自动提升为长整型(long)的警告|
|PendingDeprecationWarning|关于特性将会被废弃的警告|
|RuntimeWarning|可疑的运行时行为(runtimebehavior)的警告|
|SyntaxWarning|可疑的语法的警告|
|UserWarning|用户代码生成的警告|


### Python内建异常体系结构
---

```python
BaseException
+-- SystemExit
+-- KeyboardInterrupt
+-- GeneratorExit
+-- Exception
+-- StopIteration
+-- StandardError
|    +-- BufferError
|    +-- ArithmeticError
|    |    +-- FloatingPointError
|    |    +-- OverflowError
|    |    +-- ZeroDivisionError
|    +-- AssertionError
|    +-- AttributeError
|    +-- EnvironmentError
|    |    +-- IOError
|    |    +-- OSError
|    |         +-- WindowsError (Windows)
|    |         +-- VMSError (VMS)
|    +-- EOFError
|    +-- ImportError
|    +-- LookupError
|    |    +-- IndexError
|    |    +-- KeyError
|    +-- MemoryError
|    +-- NameError
|    |    +-- UnboundLocalError
|    +-- ReferenceError
|    +-- RuntimeError
|    |    +-- NotImplementedError
|    +-- SyntaxError
|    |    +-- IndentationError
|    |         +-- TabError
|    +-- SystemError
|    +-- TypeError
|    +-- ValueError
|         +-- UnicodeError
|              +-- UnicodeDecodeError
|              +-- UnicodeEncodeError
|              +-- UnicodeTranslateError
+-- Warning
+-- DeprecationWarning
+-- PendingDeprecationWarning
+-- RuntimeWarning
+-- SyntaxWarning
+-- UserWarning
+-- FutureWarning
+-- ImportWarning
+-- UnicodeWarning
+-- BytesWarning
```