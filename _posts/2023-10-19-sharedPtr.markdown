---
title: std::shared_ptr
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [shared_ptr]
author: miracle
mermaid: true
---

* std::shared_ptr所持有的资源可以在多个std::shared_ptr之间共享，每多一个std::shared_ptr对资源的引用，资源引用计数就会+1
* 每一个std::shared_ptr析构时，引用计数自动-1
* 最后一个std::shared_ptr析构时，如果引用计数为0，则会释放该资源
```
#include <iostream>
#include <memory>

int main() {
    //初始化方式
    std::shared_ptr<int> sp1{new int{123}}; //方式1
    std::shared_ptr<int> sp2;
    sp2.reset(new int{123}); //方式2
    std::shared_ptr<int> sp3 = std::make_shared<int>(123); //方式3， 推荐的方式

    //允许复制，每次复制，智能指针的引用计数都会原子操作+1,因此多个线程之间递增和减少引用计数是安全的，但是多个线程同时操作其指向的资源却是不安全的
    std::cout << "use count: " << sp3.use_count() << std::endl;
    std::shared_ptr<int> sp4(sp3);
    std::cout << "use count: " << sp3.use_count() << std::endl;
    sp4.reset();
    std::cout << "use count: " << sp3.use_count() << std::endl;
    {
        std::shared_ptr<int> sp5(sp3);
        std::cout << "use count: " << sp3.use_count() << std::endl;
    }
    std::cout << "use count: " << sp3.use_count() << std::endl;

    //指针大小
    int* a = new int(10);
    std::cout << "sizeof normal pointer: " << sizeof(a) << " , sizeof shared_ptr: " << sizeof(sp3) << std::endl;
}
```
输出：
```
use count: 1
use count: 2
use count: 1
use count: 2
use count: 1
sizeof normal pointer: 8 , sizeof shared_ptr: 16
```
可以看到std::shared_ptr指针的大小是普通指针大小的两倍，这是因为std::shared_ptr除了保存资源的原始指针外，还有一个指针指向其控制块,
控制块包含的内容:
* 引用计数
* 弱引用计数
* 自定义的删除器
* 分配的对象的指针
其中弱引用计数用于std::weak_ptr，弱引用计数不会影响实际的引用计数器，即引用计数归零的时候，分配的对象会被
自动删除，但是会影响控制块的删除，只要还有std::weak_ptr指向该对象，控制块就不会被删除。

std::weak_ptr是为了协助std::shared_ptr的工作引入的一个不控制资源生命周期的智能指针，是对资源的一种弱引用。
std::weak_ptr的构造和析构不会引起引用计数的增加和减少，主要用于解决std::shared_ptr的死锁问题:
例如，存在以下循环引用:
```
#include <iostream>
#include <memory>

class Node {
public:
    ~Node() { std::cout << "Deleting node\n"; }
    std::shared_ptr<Node> ptr;
};

int main() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();

    // 循环引用
    node1->ptr = node2;  // node1 引用 node2
    node2->ptr = node1;  // node2 引用 node1
}
// 程序结束时，node1 和 node2 的析构函数从未被调用，因为两者之间产生了循环引用，导致非预期的内存泄漏.
```
输出为空，也就是Node没有被释放，产生了内存泄漏
使用std::weak_ptr解此循环：
```
#include <iostream>
#include <memory>

class Node {
public:
    ~Node() { std::cout << "Deleting node\n"; }
    std::weak_ptr<Node> ptr;
};

int main() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();

    // 循环引用
    node1->ptr = node2;  // node1 引用 node2
    node2->ptr = node1;  // node2 引用 node1
}
// 程序结束时，node1 和 node2 的析构函数被正确地调用了，因为使用 std::weak_ptr 消除了循环引用.
```
输出
```
Deleting node
Deleting node
```
std::weak_ptr提供了一个expired方法用于检测资源是否仍旧有效，如果有效，就可以通过其lock方法得到一个
std::shared_ptr对象后继续操作资源:
```
#include <iostream>
#include <memory>

class Node {
public:
    ~Node() { std::cout << "Deleting node\n"; }
    std::weak_ptr<Node> ptr;
    void Print(int i) { std::cout << i << std::endl; }
};

int main() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();

    // 循环引用
    node1->ptr = node2;  // node1 引用 node2
    node2->ptr = node1;  // node2 引用 node1

    if (!node2->ptr.expired()) {
        std::shared_ptr<Node> tmp = node2->ptr.lock();
        tmp->Print(10);
    }
}
```
