---
title: algorithm
date: 2023-10-30 08:00:00 +0800
categories: [c++]
tags: [algorithm]
author: miracle
mermaid: true
---
# quick sort
```
void quickSort(vector<int>& v, int start, int end) {
    if (start >= end) { return; }
    int i = start;
    int j = end;
    int mid = start + (end - start) / 2;
    int flag = v[mid];

    while (i <= j) {
        while (v[i] < flag) { i++; }
        while (v[j] > flag) { j--; }
        if (i <= j) {
            swap(v[i], v[j]);
            i++;
            j--;
        }
    }
    if (start < j) { quickSort(v, start, j); }
    if (i < end) { quickSort(v, i, end); }
}

```

# LRU Cache
```
struct Node {
    int key;
    int value;
};

class LRU {
private:
    int m_max_size;
    list<Node> m_list;
    map<int, list<Node>::iterator> m_map;
public:
    explicit LRU(int maxSize) : m_max_size(maxSize) {}
    void put(int key, int value);
    int get(int key);
};

int LRU::get(int key) {
    auto iter = m_map.find(key);
    if (iter == m_map.end()) { return -1; }
    auto listIter = iter->second;
    Node n = (*listIter);
    m_list.erase(listIter);
    m_list.push_front(n);
    m_map[key] = m_list.begin();
    return n.value;
}

void LRU::put(int key, int value) {
    auto iter = m_map.find(key);
    if (iter != m_map.end()) {
        auto listIter = iter->second;
        m_list.erase(listIter);
    }
    Node n{key, value};
    m_list.push_front(n);
    m_map[key] = m_list.begin();
    if (m_map.size() > m_max_size) {
        int keyLast = m_list.back().key;
        m_list.pop_back();
        m_map.erase(keyLast);
    }
}
```
