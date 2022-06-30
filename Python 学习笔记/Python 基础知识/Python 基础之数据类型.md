在 Python 中我们常见的数据类型有：数字，字符串，布尔，列表，元组，集合，字典等。

### 一、数字
数字也是一类统称，里面有整数、小数等等，与之对应的还有运算符：加、减、乘、除（`+`、`-`、`*`、`/`）。
#### 1.1 数字的类型
整数的类型是 `int`  
小数的类型是 `float`
> 在 Python 里面 `**` 这个表示一个乘方，例如：`5 ** 2` 表示 5 的 2 次方。而 `=` 是用来赋值，我会在后面的笔记中单独提出 Python 中的运算符。

除了上面这两种，还有 `Decimal`、`Fraction`、`复数` 等等。

### 二、字符串
在 Python 中形如： `'......'`  或 `"......"` 的我们称为字符串。如果字符串中由特殊字符，我们需要使用 `\` 反斜杠进行转义，我们的程序才正常运行或者达到我们预期的结果。

因为 `\` 之后的字符会被转义，但我们又不希望其转义的情况，我们需要在前面加上 `r` 前缀进行说明，例如：
```python
print('C:\some\name')
# C:\some
# ame 
```
加上 `r` 前缀后：
```python
print(r'C:\some\name')
# C:\some\name
```
可以看到 '\n' 不会再被转义了，从而达到我们需要的效果。
#### 2.1 多行字符串
当我们的字符串需要格式化的实现多行字符串时，我们需要使用这种三重引号的格式，例如：`"""..."""` 或者 `'''...'''`。
```python
print("""\
Usage: thingy [OPTIONS]
     -h                        Display this usage message
     -H hostname               Hostname to connect to
""")
```
里面的内容和格式将会被原样输出。相邻的两个字符串将会被自动合并，例如：
```python
print("Hello" "Wrold")
# HelloWorld
```
数字称上一个字符串，表示重复这个字符串多少次，例如：
```python
print("Hello " * 3)
# Hello Hello Hello
```

#### 2.2 字符串索引
字符串可以看出是一个字符组成的数组，我可以通过索引的方式访问，例如：
```python
word = "你好"
print(word[0])
# 你
```
索引操作是不支持越界的，但是可以进行切片，因此，`s[:i] + s[i:]` 总是等于 `s`，大致如下：
```python
+---+---+---+---+---+---+
| P | y | t | h | o | n |
+---+---+---+---+---+---+
  0   1   2   3   4   5 
 -6  -5  -4  -3  -2  -1
```
### 三、布尔
布尔类型就是逻辑值，只有真和假，分别对应 Python 中的 `True` 和 `False`。所谓的关系运算的结果就是一个布尔值。

常见为 `False` 的情况：
```python
bool(False)
bool(None)
bool(0)
bool("")
bool(())
bool([])
bool({})
```
或者
```python
class myclass():
  def __len__(self):
    return 0

myobj = myclass()
print(bool(myobj))
```

### 四、列表
列表是一种有序，且可以更改的集合，Python 中用方括号表示。
```python
fruits = ["apple", "banana", "cherry"]
```
#### 4.1 获取项目
列表元素也是通过索引的方式获取，也同样支持切片，因此，`s[:i] + s[i:]` 总是等于 `s`。
```python
# 获取第一个
print(fruits[0])
# apple
```
#### 4.2 添加项目
```python
# 在指定位置插入
fruits.insert(1, "grape")

# 在末尾最佳
fruits.appent("orange")
```

#### 4.3 删除项目
```python
# remove 移除特定项目
fruits.remove("grape")

# 移除最后面的一个项目
fruits.pop()

# 移除索引位置的项目
del fruits[0]

# 删除整个列表
del fruits

# 清空列表
fruits.clear()
```

#### 4.4 列表其他操作
```python
# 拷贝列表
newFruits = fruits.copy()
newFruits = list(fruits)

# 合并两个列表
list1 = [1, 2 ,3]
list2 = ["a", "b", "c"]
list3 = list1 + list2
# 或者循环追加
for x in list2:
    list1.append(x)

# 将一个列表添加到另一列表尾部
list1.extend(list2)

# 统计列表项目数量
print(count(list1))

# 返回某个项目在列表中首次出现的索引位置，若不在列表里面会报错。
list1.index(1)
list1.index(-1)

# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
# ValueError: -1 is not in list
```

### 五、元组
Python 中的元组（Tuple）是一组有序，但不可更改的集合，用园括号表示。
#### 5.1 创建一个元组
```python
fruits = ("appble", "banana", "cherry")
print(fruits)
```
元组和列表的区别就是，元组的项目是不可以修改的，例如：
```python
list1 = [1, 2, 3]
list1[0] = 4

tuple1 = (1, 2, 3)
tuple1[0] = 4
# 这里会报错的，因为元组的项目不可以修改
```
> 注意切片的范围是左闭右开的的策略，如：[-4:-1] 是表示包括 -4 对应索引值，而不包括 -1 对应的索引值，即：`[-4,-1)`。

若元组只有一个项目时，需要多加一个逗号区分：
```python
thistuple = ("apple",)
print(type(thistuple))

#不是元组
thistuple = ("apple")
print(type(thistuple)
```
#### 5.2 遍历元组
```python
thistuple = ("apple", "banana", "cherry")
for x in thistuple:
  print(x)
```
#### 5.3 是否在元组中
```python
thistuple = ("apple", "banana", "cherry")
print("apple" in thistuple)
True
```
#### 5.4 其他操作
```python
# 合并两个元组
tuple1 = ("a", "b" , "c")
tuple2 = (1, 2, 3)

tuple3 = tuple1 + tuple2
print(tuple3)

# 删除元组
thistuple = ("apple", "banana", "cherry")
del thistuple
# 注意删除后，这个元组就不存在了，后面程序不可以使用

# 统计元素项目个数
thistuple.count()

# 查找元组首次出现位置，若不存在会报错
thistuple.index("apple")
```

### 六、集合
在 Python 中，集合用花括号编写，它是一种无序、无索引的。

#### 6.1 创建一个集合
```python
thisset = {"apple", "banana", "cherry"}
print(thisset)
```

#### 6.2 遍历一个集合
```python
thisset = {"apple", "banana", "cherry"}

for x in thisset:
  print(x)
```
> 注意集合是无序的，所以无法通过索引去访问集合中的项目

同样可以通过 `in` 判断是否存在集合中：
```python
thisset = {"apple", "banana", "cherry"}

print("banana" in thisset)
```
#### 6.2 其他操作
```python
thisset = {"apple", "banana", "cherry"}

# 移除一个项目
thisset.remove("apple")
# 若项目不存在会引起报错，可以用 discard()，它不会触发错误
thisset.discard("orange")
# 删除最后一个
thisset.pop()
# 清空集合
thisset.clear()
# 彻底删除集合，删除后将无法使用
del thisset

# 集合中的项目数
print(len(thisset))

# 合并两个结合
set1 = {"a", "b" , "c"}
set2 = {1, 2, 3}

set3 = set1.union(set2)
print(set3)
```
![image.png](https://api.shiqz.cn/static/upload/20220630/1656580619707655.png)

### 七、字典
字典是一个无序、可变和有索引的集合。在 Python 中，字典用花括号编写，拥有键和值。
```python
thisdict =	{
  "brand": "Porsche",
  "model": "911",
  "year": 1963
}
print(thisdict)
```
其他相关操作：
```python
# 获取键值
print(thisdict["model"])
# 等同于
print(thisdict.get("model"))

# 遍历字典
for x in thisdict:
  print(x)

# 遍历值
for x in thisdict.values():
  print(x)

# 遍历键值
for x, y in thisdict.items():
  print(x, y)

# 判断是否存在
print("model" in thisdict)

# 字典长度
print(len(thisdict))

# 添加项目
thisdict["color"] = "red"

# 删除项目
thisdict.pop("model")

# 删除最后插入的项目，（3.7 之前是随机删除）
thisdict.popitem()

# 删除指定键，必须存在不然会报错
thisdict =	{
  "brand": "Porsche",
  "model": "911",
  "year": 1963
}
del thisdict["model"]

# 删除整个字典
del thisdict

# 清空字典
thisdict.clear()

# 复制字典
dict2 = thisdict.copy()
# 等同于
dict3 = dict(thisdict)
```
> 提示字典可以嵌套

更多可以参考[菜鸟教程](https://www.w3school.com.cn/python/python_dictionaries.asp)
