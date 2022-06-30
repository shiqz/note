### 一、if else 语句
> Python 依赖缩进，使用空格来定义代码中的范围。其他编程语言通常使用花括号来实现此目的。
```python
if a < b:
    print("b is greater than a")
else:
    print("a is greater than b")
```
#### 1.1 elif
```python
a = 66
b = 66
if b > a:
  print("b is greater than a")
elif a == b:
  print("a and b are equal")
else:
    print("a is greater than b")
```

#### 1.2 and 和 or
```python
a = 200
b = 66
c = 500
if a > b and c > a:
  print("Both conditions are True")
```
```python
a = 200
b = 66
c = 500
if a > b or a > c:
  print("At least one of the conditions is True")
```
#### 1.3 pass 语句
该语句表示，这个地方暂时没有需要实现的逻辑，可以用这个占位符来防止程序报错
```python
a = 66
b = 200

if b > a:
  pass
```
### 二、While 循环语句
```python
i = 1
while i < 7:
  print(i)
  i += 1
```

#### 2.1 迭代和退出
`break` 表示结束循环，`continue` 停止本次循环，直接迭代进入下一次循环。

#### 2.2 else 语句
表示循环条件不成立才执行的模块

### 三、For 循环
`for` 循环用于迭代序列（即列表，元组，字典，集合或字符串）。例如：
```python
fruits = ["apple", "banana", "cherry"]
for x in fruits:
  print(x)
```

#### 3.1 for 与 range() 函数
`range()` 函数返回一个数字序列，从 0 开始，到 `n-1`。例如：
```python
for x in range(10):
  print(x)
```
> range(10) 不是 0 到 10 的值，而是值 0 到 9。

range函数可以接收三个参数：第一个和第二指定范围，左闭右开的原则，第三个表示从步长，也就是每次递增多少。
```python
for x in range(3, 50, 6):
  print(x)
```

#### 3.2 else 
for 循环中的 else 关键字指定循环结束时要执行的代码块：
```python
for x in range(10):
  print(x)
else:
  print("Finally finished!")
```

#### 3.3 pass 语句
for 语句不能为空，但是如果您处于某种原因写了无内容的 for 语句，请使用 pass 语句来避免错误。

