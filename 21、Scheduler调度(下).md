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

`flushWork`的逻辑很简单，只判断了一个是否帧过期，以及在这两种情况下怎么安排执行任务，执行任务的方法还是`flushTask`


```js
// 这里两个参数有点绕，需要到performWorkUntilDeadline去找
// hasTimeRemaining表示当前帧是否有剩余时间用来做任务
// initialTime表示当前帧执行任务开始的时间
function flushWork(hasTimeRemaining, initialTime) {
  // We'll need a host callback the next time work is scheduled.
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // We scheduled a timeout but it's no longer needed. Cancel it.
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  let currentTime = initialTime;
  // 检查delayTask链表里，是否有任务的startTime已经在currentTime之前了
  // 有的话则把delay任务加入到task链表里
  advanceTimers(currentTime);

  // 表示任务正在执行
  isPerformingWork = true;
  // 这里执行的错误将会直接抛出
  try {
    // 当前帧没有剩余时间
    if (!hasTimeRemaining) {
      // Flush all the expired callbacks without yielding.
      // TODO: Split flushWork into two separate functions instead of using
      // a boolean argument?
      // 没有剩余时间的情况下只处理超时时间过了的任务
      while (
        firstTask !== null &&
        firstTask.expirationTime <= currentTime &&
        !(enableSchedulerDebugging && isSchedulerPaused)
      ) {
        flushTask(firstTask, currentTime);
        currentTime = getCurrentTime();
        // 同时重新去delay任务里找有没有需要开始的任务
        advanceTimers(currentTime);
      }
    } else {
      // Keep flushing callbacks until we run out of time in the frame.
      // 当前帧还有剩余时间
      if (firstTask !== null) {
        do {
          // flushTask执行完毕后会看firstTask执行后的返回值是否是一个function
          // 如果是，会使用currentTime再次放入任务队列中
          flushTask(firstTask, currentTime);
          currentTime = getCurrentTime();
          advanceTimers(currentTime);
        } while (
          firstTask !== null &&
          // 判断是否帧的deadline已经过了
          !shouldYieldToHost() &&
          !(enableSchedulerDebugging && isSchedulerPaused)
        );
      }
    }
    // Return whether there's additional work
    // 到这里会有两种情况
    // 1. 帧时间过期，并且没有紧急任务
    // 2. 帧不过期 没有剩余任务了，也没有应该开始的delay任务
    if (firstTask !== null) {
      return true;
    } else {
      // 尝试提前触发delay任务
      if (firstDelayedTask !== null) {
        requestHostTimeout(
          handleTimeout,
          firstDelayedTask.startTime - currentTime,
        );
      }
      return false;
    }
  } finally {
    // 还回来标识
    isPerformingWork = false;
  }
}

```


`flushTask`, 执行单个任务的方法。执行以后会把执行结果保存下来，如果结果仍然是一个`function`

```js
function flushTask(task, currentTime) {
  // Remove the task from the list before calling the callback. That way the
  // list is in a consistent state even if the callback throws.
  // 对task链表的处理
  // 主要是为了把task从链表里抽离出来 ，同时链接上task原本的前后节点
  const next = task.next;
  if (next === task) {
    // This is the only scheduled task. Clear the list.
    firstTask = null;
  } else {
    // Remove the task from its position in the list.
    // 把当前要处理的task 从task链表里移除
    if (task === firstTask) {
      firstTask = next;
    }
    const previous = task.previous;
    previous.next = next;
    next.previous = previous;
  }
  task.next = task.previous = null;

  // Now it's safe to execute the task.
  // 注意这里是处理任务的callback，task本身只是一个包含了各种信息的对象
  var callback = task.callback;
  // 把任务及权重保存下来
  var previousPriorityLevel = currentPriorityLevel;
  var previousTask = currentTask;
  currentPriorityLevel = task.priorityLevel;
  currentTask = task;
  var continuationCallback;
  try {
    // 任务是否已经超时
    var didUserCallbackTimeout = task.expirationTime <= currentTime;
    // Add an extra function to the callstack. Profiling tools can use this
    // to infer the priority of work that appears higher in the stack.
    // 通过不同任务的权重，分配不同的处理方法
    switch (currentPriorityLevel) {
      case ImmediatePriority:
        continuationCallback = scheduler_flushTaskAtPriority_Immediate(
          callback,
          didUserCallbackTimeout,
        );
        break;
      case UserBlockingPriority:
        continuationCallback = scheduler_flushTaskAtPriority_UserBlocking(
          callback,
          didUserCallbackTimeout,
        );
        break;
      case NormalPriority:
        continuationCallback = scheduler_flushTaskAtPriority_Normal(
          callback,
          didUserCallbackTimeout,
        );
        break;
      case LowPriority:
        continuationCallback = scheduler_flushTaskAtPriority_Low(
          callback,
          didUserCallbackTimeout,
        );
        break;
      case IdlePriority:
        continuationCallback = scheduler_flushTaskAtPriority_Idle(
          callback,
          didUserCallbackTimeout,
        );
        break;
    }
  } catch (error) {
    throw error;
  } finally {
    currentPriorityLevel = previousPriorityLevel;
    currentTask = previousTask;
  }

  // A callback may return a continuation. The continuation should be scheduled
  // with the same priority and expiration as the just-finished callback.
  if (typeof continuationCallback === 'function') {
    var expirationTime = task.expirationTime;
    // 如果执行结果仍然是一个function, 就把task的callback改成这个执行结果
    var continuationTask = task;
    continuationTask.callback = continuationCallback;

    // Insert the new callback into the list, sorted by its timeout. This is
    // almost the same as the code in `scheduleCallback`, except the callback
    // is inserted into the list *before* callbacks of equal timeout instead
    // of after.
    // 然后把修改过的重新插入到链表里
    // 仍然按照它之前的过期时间等信息
    if (firstTask === null) {
      // This is the first callback in the list.
      firstTask = continuationTask.next = continuationTask.previous = continuationTask;
    } else {
      var nextAfterContinuation = null;
      var t = firstTask;
      // 按照权重顺序排
      do {
        if (expirationTime <= t.expirationTime) {
          // This task times out at or after the continuation. We will insert
          // the continuation *before* this task.
          nextAfterContinuation = t;
          break;
        }
        t = t.next;
      } while (t !== firstTask);
      if (nextAfterContinuation === null) {
        // No equal or lower priority task was found, which means the new task
        // is the lowest priority task in the list.
        nextAfterContinuation = firstTask;
      } else if (nextAfterContinuation === firstTask) {
        // The new task is the highest priority task in the list.
        firstTask = continuationTask;
      }

      const previous = nextAfterContinuation.previous;
      previous.next = nextAfterContinuation.previous = continuationTask;
      continuationTask.next = nextAfterContinuation;
      continuationTask.previous = previous;
    }
  }
}
```


