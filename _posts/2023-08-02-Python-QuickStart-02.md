---
layout: post
title: Python-QuickStart-02
date: 2023-08-02 16:45:30.000000000 +09:00
categories: [Python]
tags: [Python]
---


# 14 集合 Set
* 集合属于可变类型序列
* 集合中的元素不能重复

## 14.1 创建

### 方式 1： {}

```python
s = {2, 3, 4, 5, 5, 6, 7, 7}
print(s)
print(type(s)) # <class 'set'>
```

### 方式 2：内置函数 set()

```python
s = set(range(6))
print(s) # {0, 1, 2, 3, 4, 5}

# list -> set
s = set([1, 2, 3])
print(s) # {1, 2, 3}

s = set((22, False, 'hello'))
print(s) # {False, 22, 'hello'}

s = set('Python')
print(s) # {'n', 't', 'P', 'o', 'h', 'y'}

s = set({2, 3, 4, 5, 5, 6, 7, 7})
print(s) # {2, 3, 4, 5, 6, 7}

# 空 set
s = set()
print(s) # set()
```

## 14.2 判断
* in 判断是否在集合中
* not in 判断不在集合中

```python
s = {10, 20, 30, 40}

print(10 in s)     # True
print(11 not in s) # True
```

## 14.3 新增元素

* add() 方法，一次增加一个元素

```python
s.add(90)
print(s) # {40, 10, 20, 90, 30}
```

* update() 方法，至少添加一个元素

```python
s.update({11, 22})
print(s) # 传入 set {40, 10, 11, 20, 22, 90, 30}
s.update([33, 43])
print(s) # 传入 list {33, 40, 10, 11, 43, 20, 22, 90, 30}

s.update((78, 66))
print(s) # 传入 tuple {33, 66, 40, 10, 11, 43, 78, 20, 22, 90, 30}
```

## 14.4 删除操作

```python
# remove() 方法，删除指定元素，如果元素不存在抛出 KeyError
s.remove(66)
print(s) # {33, 40, 10, 11, 43, 78, 20, 22, 90, 30}

# discard() 方法，删除指定元素，如果元素不存在，不会抛出异常
s.discard(33)
s.discard(44) # 不会抛出异常
print(s) # {40, 10, 11, 43, 78, 20, 22, 90, 30}

# pop() 方法，随机删除一个元素，不能指定参数
s.pop()
print(s)

# clear() 删除所有元素
s.clear()
print(s)
```

## 14.5 集合之间的关系

```python
# 两个集合是否相等 使用 == 或 !=，
# 有相同的元素就是相等
s1 = {1, 2, 4, 3}
s2 = {2, 1, 3, 4}
print(s1 == s2) # True
print(s1 != s2) # False

# 一个集合是否是另一个的子集，调用 issubset 判断
s1 = {1, 2, 4, 3}
s2 = {2, 1}
s3 = {2, 1, 5}
print(s2.issubset(s1)) # True
print(s3.issubset(s1)) # False

# 一个集合是否是另一个的超集，调用 issuperset
print(s1.issuperset(s2)) # True
print(s1.issuperset(s3)) # False

# 两个集合 是否没有交集
print(s2.isdisjoint(s3)) # False 有交集
s4 = {6, 7}
print(s4.isdisjoint(s2)) # True 没有交集
```


## 14.6 数据操作

```python
s1 = {1, 2, 4, 3}
s2 = {2, 3, 4, 5, 6}
print('--------数据操作---------------------')
# 6.1 取交集
print(s1.intersection(s2)) # {2, 3, 4}
print(s1 & s2)             # 或者使用 & 也是取交集  {2, 3, 4}

# 6.2 取并集
print(s1.union(s2)) # {1, 2, 3, 4, 5, 6}
print(s1 | s2)      # 或者使用 | 也是取并集  {1, 2, 3, 4, 5, 6}

# 6.3 取差集 就是 A - B 集合
print('--------取差集---------------------')
print(s1.difference(s2)) # {1}
print(s1 - s2)           # 或者使用 - 取差集 {1}

# 6.4 取对称差集
print(s1.symmetric_difference(s2)) # {1, 5, 6}
print(s1 ^ s2)                     # 或使用 ^ 取对称差集 {1, 5, 6}
```

## 14.7 set 生成式
* 类似列表的生成式，就是把 [] 替换为 {}

```python
s = {x * 2 for x in range(3)}
print(s)
```