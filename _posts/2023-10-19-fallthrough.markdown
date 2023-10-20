---
title: [[fallthrough]]
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [AnnotationLabel]
author: miracle
mermaid: true
---

[[fallthrough]]作为一种注解标签，明确告知编译器和开发者该case语句被贯穿
```
#include <iostream>

void fun1() {
    std::cout << "func 1" << std::endl;
}

void fun2() {
    std::cout << "func 2" << std::endl;
}

void fun3() {
    std::cout << "func 3" << std::endl;
}

int main() {
    int funcNum = 1;
    switch (funcNum) {
        case 1:
            fun1();
            // c++ 不会像golang一样自动加上break语句，有的编译器也不告警
        case 2:
            fun2();
            [[fallthrough]]; //使用fallthrough明确告诉编译器以及开发者，该case被贯穿
        case 3:
            fun3();
            break;
        default:
            std::cout << "unknown func" << std::endl;
    }
}
```
输出:
```
func 1
func 2
func 3
```

golang中有有类似的用法:
```
package main

import "fmt"

func fun1() {
	fmt.Println("func 1")
}

func fun2() {
	fmt.Println("func 2")
}

func fun3() {
	fmt.Println("func 3")
}

func main() {
	testFuncNum := 1
	switch testFuncNum {
	case 1:
		fun1()
		fallthrough //golang会自动添加break语句，需要手动加fallthrough语句，该case语句才会贯穿
	case 2:
		fun2()
		fallthrough
	case 3:
		fun3()
	default:
		fmt.Println("unknown func")
	}
}
```
