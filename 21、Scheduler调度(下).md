上一章讲了，在调度中主要需要的几种核心方法

在`SchedulerHostConfig.default.js`里。也了解了react主要是通过浏览器帧渲染原理中的postMessage（新）或requestAnimationFrame(旧)，来保证在每一帧的空闲时间中完成js工作。

这一章我们要接着上一章的积累，看看react的调度是些什么东西

从`ReactFiberWorkLoop.js`work阶段过来，我们能看到它里面引入的是`SchedulerWithReactIntegration.js`里的很多方法，

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