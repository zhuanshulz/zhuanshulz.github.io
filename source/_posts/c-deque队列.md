---
title: c++ deque队列
date: 2024-04-30 22:32:28
tags: [C++]
---

# C++ deque双端队列

队列（Deque）是指双端队列。它泛化了队列数据结构，即可以从前端或后端执行插入和删除操作。

<!-- more -->

## 1. deque 定义

定义方法如下，其中可以输入队列中的数据类型。

```c++
std::deque<uint64_t> deque_name;
```

## 2. rbegin() rend()函数

C++ Deque rbegin()函数返回一个指向容器的最后一个元素的反向迭代器。迭代器可以递增或递减，但不能修改deque的内容。

rbegin()表示逆序开头, rend()表示逆序结尾。

![c++ rbegin function](rbegin.png)

注意：反向迭代器是从后向前迭代并向deque的开始移动的迭代器。

### 示例1

```c++
#include <iostream>
#include <deque>
using namespace std;

int main()
{
 deque deq={1,2,3,4,5};
 deque::reverse_iterator ritr=deq.rbegin();
 for(ritr=deq.rbegin();ritr!=deq.rend();++ritr)
 {
  cout<<*ritr;
  cout<<" ";
  } 
   return 0;
}
```
输出：
```c++
5 4 3 2 1 
```

### 示例2

```c++
#include <iostream>
#include <deque>
using namespace std;
int main()
{
   deque d={"java",".net","C","C++"};
   deque::reverse_iterator ritr=d.rbegin()+1;
   cout<<*ritr;
   return 0;
}
```

输出：
```c++
C 
```

## 3. push_back() push_front()函数

C++ Deque push_back()函数在deque容器的末尾添加一个新元素，并将容器的大小增加一。

C++ Deque push_front()函数在deque容器的开头插入新元素，并且容器的大小增加一。

### 示例1

```c++
#include <iostream>
#include <deque>
using namespace std;
int main()
{
    deque<int> d={10,20,30,40};
    deque<int>::iterator itr;
    d.push_back(50);
    for(itr=d.begin();itr!=d.end();++itr)
    cout<<*itr<<" ";
    return 0;
}
```
输出：
```c++
10 20 30 40 50
```

## 4. pop_back() pop_front()函数

C++ Deque pop_back()函数从deque容器中移除最后一个元素，并且deque的大小减少一个。

C++ Deque pop_front()函数从双端队列中移除第一个元素，并将容器的大小减少一个。

### 示例1

```c++
#include <iostream>
#include<deque>
using namespace std;
int main()
{
    deque<int> d={10,20,30,40,50};
    deque<int>::iterator itr;
    d.pop_front();
    for(itr=d.begin();itr!=d.end();++itr)
    cout<<*itr<<" ";
    return 0;
}
```
输出：
```c++
20 30 40 50 
```

## 5. size()函数

C++ Deque size()函数确定deque容器中的元素数量。

### 示例1

```c++
deque<int> d={1,2,3,4,5};
d.size()=5;
```

