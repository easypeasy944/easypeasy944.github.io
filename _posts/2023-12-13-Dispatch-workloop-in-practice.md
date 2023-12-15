---
title: Dispatch Workloop in practice
author: Aynur Galiev
date: 2023-12-13 20:55:00 +0300
categories: [Practice]
tags: [dispatch, swift]
pin: true
---

Couple of weeks ago I faced an unexplored API in `Dispatch` framework. As many other developers, we use `Dispatch` to organize tasks execution. But a regular task, at first glance, helped to discover a tool that can be useful in your daily work.

So here is a little background. As other complicated projects, our app had an analytics/logging system made by our own. We needed to deliver console messages, user activity events to backend after reaching a deadline or batch size limit. Besides, we had to add an ability to show cached messages in developer console for debug purposes. 

Let’s consider simple implementation below:

```swift
final class Logger {
    
    func log(_ message: String) {
         // Cache logs
    }
    
    func synchronize() {
         // Send cached logs to backend
    }
    
    func dump() {
         // Write current logs into file/dashboard/etc
    }
}
```

There is something to think about. `log`/`synchronize`/`dump` ’s code is executed on a calling thread. A caching, writing to a file or sending data over network are highly sensitive tasks from a performance perspective. Moreover, an internal properties’ access (i.e message buffer) can lead to a logic inconsistency and even cause a crash. That’s why introducing serial `DispatchQueue` and bringing all work to it looks reasonable:

```swift
final class LoggerWithQueue {
    
    private let queue = DispatchQueue(
      label: "com.easypeasy.DispatchWorkloopSandbox.syncQueue"
    )
    
    func log(_ message: String) {
        queue.async {
            // Cache logs
        }
    }
    
    func synchronize() {
        queue.async {
            // Send cached logs to backend
        }
    }
    
    func dump() {
        queue.async {
            // Write current logs into file/dashboard/etc
        }
    }
}
```

As we know, a serial queue executes tasks in **FIFO** (first-in-first-out) order. It means we can’t sync logs until a dump is done. A data sync is much more important according to business requirements. So, we need enhanced `DispatchQueue` that executes one task at a time but takes into account its priority.

`DispatchWorkloop` is a subclass of `DispatchQueue` that was introduced in iOS 12. It is a priority ordered queue. `DispatchWorkloop` looks for a task with higher priority between work items execution. Thus, `DispatchWorkloop` can reorder tasks queue to provide better performance. 

Let’s tweak code above a bit:

```swift
final class LoggerWithWorkloop {
    
    private let queue = DispatchWorkloop(
			label: "com.easypeasy.DispatchWorkloopSandbox.syncQueue"
    )
    
    func log(_ message: String) {
        let workItem = DispatchWorkItem(qos: .default) {
            // Cache logs
        }
        queue.async(execute: workItem)
    }
    
    func synchronize() {
        let workItem = DispatchWorkItem(qos: .userInitiated) {
            // Send cached logs to backend
        }
        queue.async(execute: workItem)
    }
    
    func dump() {
        let workItem = DispatchWorkItem(qos: .utility) {
            // Write current logs into file/dashboard/etc
        }
        queue.async(execute: workItem)
    }
}
```
The code guarantees that logs dump won’t block data sync whereas incoming log wait until sync is done. So, we secure ourselves from multithreading issues and make code execution more accurate and efficient. 

[Dispatch Workloop's documentation]: https://developer.apple.com/documentation/dispatch/workloop
