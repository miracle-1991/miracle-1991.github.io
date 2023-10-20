---
title: emplace_back
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [emplace]
author: miracle
---

旧的push_back采用的是拷贝赋值的方式，会导致不必要的构造和析构， 比如：
```
#include <iostream>
#include <list>

class Test {
public:
    Test(int a, int b, int c) : ma(a),mb(b),mc(c){
        std::cout << "Test constructed" << std::endl;
    }
    ~Test() {
        std::cout << "Test destructed" << std::endl;
    }
    Test(const Test& rhs) {
        if (this == &rhs) {
            return;
        }
        ma = rhs.ma;
        mb = rhs.mb;
        mc = rhs.mc;
        std::cout << "Test Copy-constructed" << std::endl;
    }
private:
    int ma;
    int mb;
    int mc;
};

int main() {
    std::list<Test> collections;
    Test t{1,2,3};
    collections.push_back(t);
    return 0;
}
```
输出为：
```
Test constructed
Test Copy-constructed
Test destructed
Test destructed
```
t的构造、拷贝、析构其实完全没有必要， 采用emplace_back,可以在容器的尾部原位构造元素：
```
int main() {
    std::list<Test> collections;
    collections.emplace_back(1,2,3);
    return 0;
}
```
输出为:
```
Test constructed
Test destructed
```
仅仅需要在容器尾部构造一次，这种效率的提升在循环时尤其重要。
