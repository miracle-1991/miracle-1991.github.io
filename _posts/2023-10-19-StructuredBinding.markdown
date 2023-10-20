---
title: Structured Binding Declaration
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [Binding]
author: miracle
---

结构化绑定是c++17引入的一种新的语法形式，可以在一次声明中引入多个变量，在使用容器、pair、tuple时能大幅简化代码。

比如：
```
#include <iostream>
#include <string>
#include <map>
#include <tuple>

using TupleRET = std::tuple<std::string, std::string, int, int, std::string>;
TupleRET returnTupleFunc() {
    return TupleRET{"Tom", "123456",0,25,"Pudong Street"};
}

int main() {
    // map, 插入pair，不使用结构化绑定
    std::map<std::string, std::string> cities;
    cities["beijing"] = "0";
    auto insertResult = cities.insert(std::pair<std::string, std::string>("shanghi", "2")); //pair的第一个值是元素的迭代器，第二个值是布尔值，是否插入成功
    std::cout << "Is insertion successful : " << (insertResult.second ? "true" : "false") << std::endl;
    std::cout << "Element Key: " << insertResult.first->first << ", value: " << insertResult.first->second << std::endl;

    // map, 插入元素，使用结构化绑定, 按值传递
    auto [iterator, inserted] = cities.insert(std::make_pair("tianjin", "3"));
    std::cout << "Is insertion successful : " << (inserted ? "true" : "false") << std::endl;
    std::cout << "Element Key: " << iterator->first << ", value: " << iterator->second << std::endl;

    // tuple, 不使用结构化绑定, 按值传递
    TupleRET userInfo = returnTupleFunc();
    std::string userName = std::get<0>(userInfo);
    std::string passWord = std::get<1>(userInfo);
    int gender = std::get<2>(userInfo);
    int age = std::get<3>(userInfo);
    std::string address = std::get<4>(userInfo);
    std::cout << "name: " << userName << ", pwd: " << passWord << ", gender: " << gender << " , age:" << age << " ,address:"<< address << std::endl;

    // tuple，使用结构化绑定，大幅度简化代码，能做到见名知意
    auto [name, pwd, gen, ag, addr] = returnTupleFunc();
    std::cout << "name: " << name << ", pwd: " << pwd << ", gender: " << gender << " , age:" << ag << " ,address:"<< addr << std::endl;
}
```
