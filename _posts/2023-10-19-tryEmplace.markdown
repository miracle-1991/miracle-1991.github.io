---
title: try_emplace
date: 2023-10-19 08:00:00 +0800
categories: [c++]
tags: [empalce, c++17]
author: miracle
---

try_emplace是c++17引入的新的方法，可以检测容器中的key是否存在：
* 存在，什么也不做，也就是不做更新操作
* 不存在，才插入
可以简化代码，不用再手动find并判断find的结果是否是end()
比如：
```
#include <iostream>
#include <map>
#include <memory>

class ChatDialog {
public:
    void active() { std::cout << "active" << std::endl; }
};


std::map<int64_t, std::unique_ptr<ChatDialog>> m_chatDialogs;

// 一般写法, 先检查是否存在，不存在则插入，存在则调用
void onClick(int64_t userID) {
    auto target = m_chatDialogs.find(userID);
    if (target == m_chatDialogs.end()) {
        auto pChatDiag = std::make_unique<ChatDialog>();
        m_chatDialogs.insert(std::make_pair(userID, std::move(pChatDiag)));
        pChatDiag->active();
    } else{
        target->second->active();
    }
}

// 使用try_emplace,检测key是否存在，如果存在，什么也不做,只调用，如果不存在，则插入成功
void onClick2(int64_t userID) {
    auto [iter, inserted] = m_chatDialogs.try_emplace(userID);
    if (inserted) {
        auto pChatDiag = std::make_unique<ChatDialog>();
        iter->second = std::move(pChatDiag);
    }
    iter->second->active();
}

int main() {
    int64_t useID = 10086;
    onClick(useID);
    onClick2(useID);
}
```
