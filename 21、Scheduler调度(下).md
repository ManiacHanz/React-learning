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
    if (firstTask === null && firstDelayedTask === newTask) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    insertScheduledTask(newTask, expirationTime);
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```