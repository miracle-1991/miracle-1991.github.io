---
title: std::unique_ptr
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [shared_ptr, c++11]
author: miracle
mermaid: true
---

std::unique_ptr脱胎于c++98/03的std::auto_ptr，用于替代std::auto_ptr。
std::unique_ptr对其持有的内存具有唯一拥有权，在其销毁时会自动释放持有的堆内存。
std::unique_ptr的大小与普通指针大小一样，在64位机器上占8个字节，在32位机器上占4个字节。
使用例子：
```
#include <iostream>
#include <memory>

class Test {
public:
    Test() { std::cout << "constructor" << std::endl; }
    ~Test() { std::cout << "distructor" << std::endl; }
    void Print(int i) { std::cout << i << std::endl; }
};

int main() {
    //初始化方式
    std::unique_ptr<int> sp1{new int{123}}; //方式1
    std::unique_ptr<int> sp2;
    sp2.reset(new int{123}); //方式2
    std::unique_ptr<int> sp3 = std::make_unique<int>(123); //方式3， 推荐的方式

    //禁止复制，只能移动
    //    std::unique_ptr<int> sp4(sp3);  无法编译：Call to implicitly-deleted copy constructor of 'std::unique_ptr<int>'
    //    std::unique_ptr<int> sp5 = sp3; 无法编译：Call to implicitly-deleted copy constructor of 'std::unique_ptr<int>'

    //不仅可以持有一个堆对象，还能持有一组堆对象
    std::unique_ptr<Test[]> sp6(new Test[3]);
    for (int i = 0; i < 3; i++) {
        sp6[i].Print(i);
    }

    //指针大小
    int* a = new int(10);
    std::cout << "sizeof normal pointer: " << sizeof(a) << " , sizeof unique_ptr: " << sizeof(sp6) << std::endl;
}
```
输出:
```
constructor
constructor
constructor
0
1
2
sizeof normal pointer: 8 , sizeof unique_ptr: 8
distructor
distructor
distructor
```
