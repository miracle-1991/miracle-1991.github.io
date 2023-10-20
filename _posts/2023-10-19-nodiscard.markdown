---
title: [[nodiscard]]
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [AnnotationLabel]
author: miracle
---

[[nodiscard]]标签用于告知编译器此函数的返回值不可被使用者忽略，必须被使用，常用于：
* 返回FD
* 返回new或者allocate的结果

用法：
```
#include <iostream>

[[nodiscard]] int fun1() {
    std::cout << "func 1" << std::endl;
     return 1;
}

int main() {
    fun1();
}
```
测试程序中没有使用func1的返回值，编译器对此进行了告警:
```
main.cpp:9:5: warning: ignoring return value of function declared with 'nodiscard' attribute [-Wunused-result]
    fun1();
    ^~~~
```
