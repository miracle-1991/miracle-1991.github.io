---
title: [[deprecated]]
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [AnnotationLabel]
author: miracle
---

[[deprecated]]注解标签用于告诉使用者该函数或者该类型已经被弃用。当使用这些被弃用的函数或者类型时，编译器会告警。

使用示例：
```
#include <iostream>

[[deprecated("use your func")]] void funcX() {
    std::cout << "funcx" << std::endl;
}

int main() {
    funcX();
}
```
编译时会遇到告警:
```
'funcX' is deprecated: use your func
```

golang中有类似的方法标记某个变量或者函数是废弃的:
```
package main

import (
	"fmt"
)

// Deprecated
func printMsg() {
	fmt.Printf("i am deprecated\n")
}

func main() {
	printMsg()
}
```
在IDE中printMsg()会显示成~~printMsg~~,并提示该函数是deprecated
