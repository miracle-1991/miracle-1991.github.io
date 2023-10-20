---
title: initializer_list
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [initializer_list]
author: miracle
mermaid: true
---

std::initializer_list<T>是c++11版本引入的模板对象，用来统一和规范c++对象初始化的方式，T可以是任意的基础数据类型
也可以是复杂的自定义类型

使用方式如下:
* 头文件
```
#include <iostream>
#include <initializer_list>
#include <vector>

class A
{
private:
    std::vector<int> m_vecIntegers;
public:
    A(std::initializer_list<int> integers);
    ~A();
    void append(std::initializer_list<int> integers) {
        m_vecIntegers.insert(m_vecIntegers.end(), integers.begin(), integers.end());
    }

    void print() {
        size_t size = m_vecIntegers.size();
        for (size_t i = 0; i < size; i++) {
            std::cout << m_vecIntegers[i] << std::endl;
        }
    }
};

A::A(std::initializer_list<int> integers)
{
    m_vecIntegers.insert(m_vecIntegers.end(), integers.begin(), integers.end());
}

A::~A() {}
```
* 测试代码
```
int main() {
    A a{1,2,3,4};
    a.print();

    a.append({5,6,7});
    a.print();
}
```
