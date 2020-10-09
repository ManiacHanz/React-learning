上一章讲了，在调度中主要需要的几种核心方法

在`SchedulerHostConfig.default.js`里。也了解了react主要是通过浏览器帧渲染原理中的postMessage（新）或requestAnimationFrame(旧)，来保证在每一帧的空闲时间中完成js工作。

这一章我们要接着上一章的积累，看看react的调度是些什么东西

从`ReactFiberWorkLoop.js`work阶段过来，我们能看到它里面引入的是`SchedulerWithReactIntegration.js`里的很多方法，而其中的许多方法都是从`scheduler`引进来。这个文件本身只是根据任务权重做了下集成

```js
import {
  scheduleCallback,
  cancelCallback,
  getCurrentPriorityLevel,
  runWithPriority,
  shouldYield,
  requestPaint,
  now,
  NoPriority,
  ImmediatePriority,
  UserBlockingPriority,
  NormalPriority,
  LowPriority,
  IdlePriority,
  flushSyncCallbackQueue,
  scheduleSyncCallback,
} from './SchedulerWithReactIntegration';

// The scheduler is imported here *only* to detect whether it's been mocked
import * as Scheduler from 'scheduler';
```

根据线索，我们很容易能找到`schedule`文件，就从他里面的一些方法来看吧，首先是`scheduleCallback`

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 通过performance.now()获取当前时间
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;
  // option 里传入了delay ,一般是suspence组件之类，这里先不看
  // 在fiber计算里如果有特殊情况写入了timeout或者delay，则按规则计算startTime和timeout
  // 否则按照常规方法计算
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }

  // 主要的目的是计算expirationTime
  var expirationTime = startTime + timeout;

  // 生成默认task对象
  var newTask = {
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    next: null,
    previous: null,
  };

  if (startTime > currentTime) {
    // This is a delayed task.
    // 只有延迟任务才会有这个情况，就直接加入到延迟任务队列里
    insertDelayedTask(newTask, startTime);
    // firstTask 是整个task链表的第一个
    if (firstTask === null && firstDelayedTask === newTask) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      // 任务队列里只有delay任务 直接触发
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 插入任务队列
    insertScheduledTask(newTask, expirationTime);
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // isHostCallbackScheduled 表示在触发callback
    // 在requestHostCallback改为true 在flushWork里置位false
    // isPerformingWork 表示任务正在进行 在flushWork开始时为true，结束时为false
    // 所以这里如果没有任务正在执行 就会调用触发flushWork
    // 也就是我们上一章看到的，通过postMessage或者rAF触发flushWork
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}

// 按照expirationTime的优先级，把任务插入到任务链表里的正确位置
function insertScheduledTask(newTask, expirationTime) {
  // Insert the new task into the list, ordered first by its timeout, then by
  // insertion. So the new task is inserted after any other task the
  // same timeout
  if (firstTask === null) {
    // This is the first task in the list.
    firstTask = newTask.next = newTask.previous = newTask;
  } else {
    var next = null;
    var task = firstTask;
    do {
      if (expirationTime < task.expirationTime) {
        // The new task times out before this one.
        next = task;
        break;
      }
      task = task.next;
    } while (task !== firstTask);

    if (next === null) {
      // No task with a later timeout was found, which means the new task has
      // the latest timeout in the list.
      next = firstTask;
    } else if (next === firstTask) {
      // The new task has the earliest expiration in the entire list.
      firstTask = newTask;
    }

    var previous = next.previous;
    previous.next = next.previous = newTask;
    newTask.next = next;
    newTask.previous = previous;
  }
}
```

可以看到`scheduleCallback`主要是做一个插入队列的工作，在没有任务执行的情况下，通过`postMessage`或者`rAF`触发`flushWork`。


```js
function flushWork(hasTimeRemaining, initialTime) {
  // We'll need a host callback the next time work is scheduled.
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // We scheduled a timeout but it's no longer needed. Cancel it.
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  let currentTime = initialTime;
  advanceTimers(currentTime);

  isPerformingWork = true;
  try {
    if (!hasTimeRemaining) {
      // Flush all the expired callbacks without yielding.
      // TODO: Split flushWork into two separate functions instead of using
      // a boolean argument?
      while (
        firstTask !== null &&
        firstTask.expirationTime <= currentTime &&
        !(enableSchedulerDebugging && isSchedulerPaused)
      ) {
        flushTask(firstTask, currentTime);
        currentTime = getCurrentTime();
        advanceTimers(currentTime);
      }
    } else {
      // Keep flushing callbacks until we run out of time in the frame.
      if (firstTask !== null) {
        do {
          flushTask(firstTask, currentTime);
          currentTime = getCurrentTime();
          advanceTimers(currentTime);
        } while (
          firstTask !== null &&
          !shouldYieldToHost() &&
          !(enableSchedulerDebugging && isSchedulerPaused)
        );
      }
    }
    // Return whether there's additional work
    if (firstTask !== null) {
      return true;
    } else {
      if (firstDelayedTask !== null) {
        requestHostTimeout(
          handleTimeout,
          firstDelayedTask.startTime - currentTime,
        );
      }
      return false;
    }
  } finally {
    isPerformingWork = false;
  }
}

```