---
layout: post
title: Python-QuickStart-01
date: 2023-07-30 16:45:30.000000000 +09:00
categories: [Python]
tags: [Python]
---

# 1 print 函数

```python
fp = open('~/test.txt', 'a+')
print("hello world", file=fp)
fp.close()

print('hello', 'world', 'pyth')


# end 参数自定义
for i in range(3):
    print(i, end='\t')

```

# 2 转义字符

```python
print("hello\nworld")
print("hello/tworld")     # hello/tworld
print("helloooo\tworld")  # helloooo  world
print("hello\rworld")     # world
print("hello\bworld")     # \b 是一个退格 将 o 退没, hellworld
```

# 3 元字符 
* 不希望字符串中的字符被转义就在字符串前加上 r 或 R
* 最后一个字符不能是反斜线

```python
print(r"hello\nworld") # hello\nworld
```

# 4 python 中的保留字

```python
import keyword
print(keyword.kwlist)
```

# 5 变量
* 可以理解为变量指向一个 object，这个 object 有 id、type、value 三个值

```python
name = "Maria" # name 指向 id 内存地址
print("标识", id(name)) # 内存地址
print("类型", type(name))
print("值", name)
```


# 6 数据类型
* 6.1 布尔类型 True or False
* True 转为整数为 1，False 转为整数为 0

## 6.2 整数 int

```python
print(0b1101) # 二进制
print(0o10)  # 八进制
```


## 6.3 浮点数

```python
from decimal import Decimal
print(Decimal('1.1') + Decimal('2.2'))  # 3.3
```

# 7 字符串
* 字符串为不可变的字符序列

```python
# str1 和 str2 的 id 一样
str1 = '人生苦短，我用 Python'
str2 = "人生苦短，我用 Python"
str3 = """人生苦短，
我用 Python"""
print(str1, id(str1))  # 人生苦短，我用 Python 140658434963696
print(str2, id(str2))  # 人生苦短，我用 Python 140658434963696
print(str3, id(str3))  # 人生苦短，我用 Python 140658435059248
```




# 8 类型转换
* str() 将其他类型转为字符串
* int() 将其他类型转为 int 类型

```python
age = 20
print("我叫"+name+'今年'+str(age)+'岁')

print(int(name)) 使用 int 时，字符串必须是整数字符串，否则报错
```


# 9 pass 语句
* pass 语句可用于占位保证编译通过

```python
if 10 > 1:
    pass
else:
    pass
```

# 10 控制语句

## for else

```python
for i in range(3):
    pwd = input('请输入密码')
    if pwd == '888':
        break
    else:
        print('密码不正确')
else:
    print('抱歉，三次密码输入错误')
```

## while else

```python
a = 0
while a < 3:
    pwd = input('请输入密码')
    if pwd == '888':
        break
    else:
        print('密码不正确')
    a += 1
else:
    print('抱歉，三次密码输入错误')
```


# 11 列表
* 列表是存储多个变量的引用，lst = ['hello', 'world', 98]

> a = 10 变量存储的是一个变量的引用
{: .prompt-info }



## 11.1 创建列表的方式
1. 使用中括号
2. 使用内置函数 list

```python
# 方式一：使用中括号
list1 = ['hello', 'world', 98, 'hello']
print(id(list1))


# 方式二：使用内置函数 list
list2 = list(['hello', 'world', 98])

print(id(list2))
```

## 11.2 index() 方法
* index() 返回列表中第一个匹配的元素的 index，如果查找的对象在列表中不存在会抛出异常

```python
print(list1.index('hello'))

# 在索引 [1, 4) 范围内查找 hello
print(list1.index('hello', 1, 4))
```


## 11.3 索引
* 如果列表有 N 个元素
1. 正向索引是 0 到 N-1
2. 逆向索引是 -N 到 -1
3. 指定的索引不存在则抛出异常

```python
list1 = ['hello', 'world', 98, 'hello', 'world', 123]
print(list1[2])  # 98

print(list1[-3]) # hello
```


## 11.4 切片
* 格式 [start : stop : step]
* 切片的结果是原列表的拷贝
* 切片的范围是 [start, stop)
* step 默认是 1
* 切片里边的对象是浅拷贝

```python
lst = [10, 20, 30, 40, 50, 60, 70, 80]
print(lst[1:6])
print('原列表', id(lst))

lst2 = lst[1:6]
# 切片里边的对象是浅拷贝

print('-----step 为正----------')
# step 为正
# 切片第一个元素是列表第一个元素
# 切片最后一个元素是列表最后一个元素
print('1:6', id(lst2))

print(id(lst[1]))
print(id(lst2[0]))

# 默认 start 是 0，stop = N
lst2 = lst[1:6]

print('-----step 为负----------')
# step 为负，可以理解为 reverse 操作，即从后往前遍历
# 切片第一个元素是列表最后一个元素
# 切片最后一个元素是列表第一个元素
# lst[::-1] == lst[7::-1]
print(lst[::-1])
print(lst[7::-1])
```


## 11.5 列表判断
* in 表示存储于列表中，存在返回 True, 不存在返回 False
* not in 判断不在列表中

```python
print('k' in 'python')
print('k' not in 'python')
```

## 11.6 遍历列表

```python
lst = [10, 20, 30, 40, 50, 60, 70, 80]

for item in lst:
    print(item)
```

## 11.7 列表元素添加
### 1 append 向列表末尾添加元素 （常用）

```python
lst.append(90)
print(lst) # [10, 20, 30, 40, 50, 60, 70, 80, 90]
```

### 2 extend 向列表末尾添加至少一个元素（相当于 swift 的 appendFromArray）

```python
lst = [1, 2, 3]
lst1 = [4, 5]
lst.extend(lst1)
print(lst) # [1, 2, 3, 4, 5]
```

### 3 insert 在任意位置插入元素

```python
# 在 index=1 位置插入 9
lst.insert(1, 9)
print(lst) # [1, 9, 2, 3, 4, 5]
```

### 4 切片，在任意位置插入多个元素

```python
lst3 = [True, False, 'ho']
# 从 index=1 开始切，[1, 3) 位置替换为 lst3 的内容
lst[1:3] = lst3
print(lst) # [1, True, False, 'ho', 3, 4, 5]
```

## 11.8 列表删除元素
### 1 remove 一次删除一个元素，重复的元素只删除第一个元素
* 元素不存在抛出异常

```python
lst = [10, 20, 30, 40, 50, 60, 30]
lst.remove(30) # [10, 20, 40, 50, 60, 30]
print(lst)
```

### 2 pop 删除指定索引位置的元素
* 指定索引不存在抛出异常，out of bounds
* 不指定索引删除最后一个元素

```python
lst.pop(1) # 删除索引为 1 的元素
print(lst) # [10, 40, 50, 60, 30]

lst.pop()
print(lst) # [10, 40, 50, 60]
```

### 3 切片，一次至少删除一个元素
* 注意：切片可能会产生新的列表对象

```python
print('---------注意：切片会产生新的列表对象---------------')
# lst = lst[1:3] # [40, 50]
print(lst)

print('---------切片不产生新的列表对象---------------')
# 使用空列表替换切的位置
lst[1:3] = []
print(lst)
```

### 4 clear 清空列表所有元素

```python
lst.clear()
print(lst)
```

### 5 del 删除定义的变量

```python
del lst
print(lst) # 删除后再打印会报错，NameError: name 'lst' is not defined
```

## 11.9 列表元素修改

### 1 通过索引修改

```python
lst = [10, 20, 30, 40, 50, 60, 30]
lst[1] = 33
print(lst) # [10, 33, 30, 40, 50, 60, 30]
```

### 2 通过切片修改
* 赋值的超过了切片的 count 也可以

```python
lst[1:4] = [11, 22, 33, 44]
print(lst) # [10, 11, 22, 33, 44, 50, 60, 30]
```

## 11.10 列表的排序

### 1 调用 sort 函数，不产生新列表 地址不变
* reverse=True 表示降序，默认不传是升序

```python
lst = [33, 22, 55, 99, 66, 88, 11, 44, 77]
# 降序排列
print(id(lst))
lst.sort(reverse=True)
print(id(lst))
print(lst)
```

### 2 使用内置函数 sorted，产生一个新列表对象

```python
lst = [33, 22, 55, 99, 66, 88, 11, 44, 77]
lst1 = sorted(lst)
# reverse 为 True 为降序
lst1 = sorted(lst, reverse=True)
print(lst1)
print(id(lst))
print(id(lst1))
```

## 11.11 生成列表的公式
* `[表示列表元素表达式 for 自定义变量 可迭代对象]`
* 表示列表元素表达式中，通常包括自定义变量

```python
lst = [x for x in range(1, 10)]
print(lst)
```

# 12 字典



