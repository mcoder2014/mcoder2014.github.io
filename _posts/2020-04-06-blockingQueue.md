---
layout:     post
title:      "多线程开发"
subtitle:   "阻塞队列"
date:       2020-04-06
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 服务器
    - 后台开发
    - 多线程
---

阻塞队列是多线程中常用的数据结构，对于实现多线程之间的数据交换、同步等有很大作用。

阻塞队列常用于**生产者和消费者的场景**，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。简而言之，阻塞队列是生产者用来存放元素、消费者获取元素的容器。

考虑下，这样一个多线程模型，程序有一个主线程 master 和一些 worker 线程，master 线程负责接收到数据，给 worker 线程分配数据，worker 线程取得一个任务后便可以开始工作，如果没有任务便阻塞住，节约 cpu 资源。

- master 线程  （生产者）：负责往阻塞队列中塞入数据，并唤醒正在阻塞的 worker 线程。
- workder 线程（消费者）：负责从阻塞队列中取数据，如果没有数据便阻塞，直到被 master 线程唤醒。

那么怎样的数据结构比较适合做这样的唤醒呢？显而易见，是条件变量，在 c++ 11 中，stl 已经引入了线程支持库。

# C++11 中条件变量
条件变量一般与一个 **互斥量** 同时使用，使用时需要先给互斥量上锁，然后条件变量会检测是否满足条件，如果不满足条件便会暂时释放锁，然后阻塞线程。
 c++ 11使用方法主要如下：
```
#include <mutex>
#include <condition_value>
// 互斥量与条件变量
std::mutex m_mutex;
std::condition_value m_condition;

// 请求信号的一方
std::unique_lock<std::mutex> lock(mutex);
while(xxx)
{
    // 这里会先释放 lock，
    // 如果有信号唤醒的话，会重新加锁。
    m_condition.wait(lock);
}

// 发送消息进行同步的一方
{
    std::unique_lock<std::mutex> lock(mutex);
    // 唤醒其他正在 wait 的线程
    m_condition.notify_all();
}
```

# 用 C++11 实现阻塞队列
我们使用条件变量包装 STL 中的 queue 就可以实现阻塞队列功能，如果有兴趣，甚至可以自己实现一个效率更高的队列数据结构。

我们先假设一下阻塞队列需要如下接口：
1. push 将一个变量塞入队列；
2. take 从队列中取出一个元素；
3. size 查看队列有多少个元素；
```
template <typename T>
class BlockingQueue
{
public:
    BlockingQueue();
    void push(T&& value);
    T take();
    size_t size() const;
    
private:
    // 实际使用的数据结构队列
    std::queue<T> m_data;

    // 条件变量
    std::mutex m_mutex;
    std::condition_variable m_condition;
};
```

push 一个变量时，我们需要先加锁，加锁成功后才可以压入变量，这是为了线程安全。压入变量后，就可以发送信号通知正在阻塞的条件变量。
```
    void push(T&& value)
    {
        // 往队列中塞数据前要先加锁
        std::unique_lock<std::mutex> lock(m_mutex);
        m_data.push(value);
        // 唤醒正在阻塞的条件变量
        m_condition.notify_all();
    }
```

take 一个变量时，就要有些不一样：
1. 先加锁，加锁成功后，如果队列不为空，可以直接取数据，不需要 wait；
2. 如果队列为空呢，则 wait 等待，直到被唤醒；
3. 考虑特殊情况，唤醒后队列依然是空的……

```
    T take()
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        while(m_data.empty())
        {
            // 等待
            m_condition.wait(lock);
        }
        assert(!m_data.empty());
        T value(std::move(m_data.front()));
        m_data.pop();

        return value;
    }
```

总结下，代码如下：
```
#ifndef BLOCKINGQUEUE_H
#define BLOCKINGQUEUE_H

#include <queue>
#include <mutex>
#include <condition_variable>
#include <assert.h>

template <typename T>
class BlockingQueue
{
public:
    BlockingQueue()
        :m_mutex(),
          m_condition(),
          m_data()
    {
    }

    // 禁止拷贝构造
    BlockingQueue(BlockingQueue&) = delete;

    ~BlockingQueue()
    {
    }

    void push(T&& value)
    {
        // 往队列中塞数据前要先加锁
        std::unique_lock<std::mutex> lock(m_mutex);
        m_data.push(value);
        m_condition.notify_all();
    }

    void push(const T& value)
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        m_data.push(value);
        m_condition.notify_all();
    }

    T take()
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        while(m_data.empty())
        {
            m_condition.wait(lock);
        }
        assert(!m_data.empty());
        T value(std::move(m_data.front()));
        m_data.pop();

        return value;
    }

    size_t size() const
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        return m_data.size();
    }
private:
    // 实际使用的数据结构队列
    std::queue<T> m_data;

    // 条件变量的锁
    std::mutex m_mutex;
    std::condition_variable m_condition;
};
#endif // BLOCKINGQUEUE_H
```

## 实验代码
我们写个简单的程序实验一下，下面程序有 一个 master 线程，5个 worker 线程，master线程生成一个随机数，求 0-随机数 的和。

```c++
#include <iostream>
#include <thread>
#include <mutex>
#include <random>

#include <windows.h>

#include <blockingqueue.h>
using namespace std;

int task=100;
BlockingQueue<int> blockingqueue;
std::mutex mutex_cout;

void worker()
{
    int value;
    thread::id this_id = this_thread::get_id();
    while(true)
    {
        value = blockingqueue.take();
        uint64_t sum = 0;
        for(int i = 0; i < value; i++)
        {
            sum += i;
        }

        // 模拟耗时操作
        Sleep(100);

        std::lock_guard<mutex> lock(mutex_cout);
        std::cout << "workder: " << this_id << " "
                  << __FUNCTION__
                  << " line: " << __LINE__
                  << " sum: " << sum
                  << std::endl;
    }
}

void master()
{
    srand(time(nullptr));
    for(int i = 0; i < task; i++)
    {
        blockingqueue.push(rand()%10000);
        printf("%s %d %i\n",__FUNCTION__, __LINE__, i);
        Sleep(20);
    }
}

int main()
{
    thread th_master(master);
    std::vector<thread> th_workers;
    for(int i =0; i < 5; i++)
    {
        th_workers.emplace_back(thread(worker));
    }

    th_master.join();
    return 0;
}

```

从输出结果可以看出，master 线程将任务分配给了正在空闲的 worker 线程，具体是哪个线程就看操作系统的随机调度了。
```
master 46 5
worker: 3 worker line: 34 sum: 20998440
master 46 6
worker: 7 worker line: 34 sum: 3308878
master 46 7
worker: 4 worker line: 34 sum: 34598721
master 46 8
worker: 6 worker line: 34 sum: 1563796
master 46 9
worker: 5 worker line: 34 sum: 27978940
```

# Reference
1. [条件变量](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_for)