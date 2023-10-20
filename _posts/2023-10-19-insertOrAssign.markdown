---
title: insert_or_assign
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [map]
author: miracle
mermaid: true
---

在map中，如果需要先判断是否存在，然后：
* 存在，则更新
* 不存在，则插入

可以使用map[key]=val的方式，也可以使用insert_or_assign，区别在于返回值，insert_or_assign会返回一个pair，包含了：
* 元素的迭代器
* 是否插入了新值

在需要使用这种返回值的地方尤其有用，避免了再去查找某个元素是否存在：
```
#include <iostream>
#include <map>

int main() {
    std::map<std::string, int> mapUseAge{
      {"ALEX", 45}, {"JOHN", 25}
    };
    auto [iter, inserted] = mapUseAge.insert_or_assign("JOHN", 26);
    std::cout << "insert JOHN " << inserted << std::endl;
}
```
输出:
```
insert JOHN 0
```
