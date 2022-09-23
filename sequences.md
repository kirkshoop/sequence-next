---
title: "Asynchronous Value Sequences"
subtitle: "Draft Proposal"
document: D0000R0
date: today
audience:
  - "SG1 - parallelism"
author:
  - name: Kirk Shoop
    email: <kirk.shoop@gmail.com>
toc: true
---

Introduction
============

This was the end goal all along.

Sender/Receiver, as described in [@P2300R5], can represent a single asynchronous value. A generator of `range<Sender>` can represent a set of asynchronous values, where the range generates each sender synchronously. This paper is about a Sender that can asynchronously send `0-N` values and some of the algorithms that operate on an asynchronous value sequence.

```cpp
sync_wait(generate_each(&::getchar)
   | take_while([](char v) { return v != '0'; })
   | filter_each(std::not_fn(&::isdigit))
   | then_each(&::toupper)
   | then_each([](char v) { std::cout << v; })
   | ignore_all());
```

```cpp
auto counters = std::map<std::thread::id, std::ptrdiff_t>{};
sync_wait(itoas(1, 3000000)
   | fork(pool)
   | then_each([](int v){ return std::this_thread::get_id();})
   | on_each(single) // join
   | then_each([&counters](std::thread::id tid) { ++counters[tid];})
   | ignore_all()
   | then([&counters](){for(auto& c : counters){
    std::cout << c.first << " : " << c.second;
   }}));
```
