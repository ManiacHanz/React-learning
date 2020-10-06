

> 有了MessageChannel的加持，整个调度其实是一个发布-订阅模式，通过onmessage来触发调度

有了上一章的前置知识，我们知道了 `requestIdleCallback`虽然可以天然在浏览器帧渲染的空闲时间进行回调工作，但是有许多天然限制
1. 不能保证每一帧都有空闲时间执行回调
2. 默认只有20fps，即每一帧定为50ms，远低于60fps的需求
3. 做复杂回调后容易超时，造成掉帧
4. 没有polyfill，低浏览器不好兼容

所以react没有用`requestIdleCallback`

当然，具体用的什么方案，我们先看`SchedulerHostConfig.default.js`这个文件

先放一段关于这个文件的注释

> The DOM Scheduler implementation is similar to requestIdleCallback. It
works by scheduling a requestAnimationFrame, storing the time for the start
of the frame, then scheduling a postMessage which gets scheduled after paint.
Within the postMessage handler do as much work as possible until time + frame
rate. By separating the idle call into a separate event tick we ensure that
layout, paint and other browser work is counted against the available time.
The frame rate is dynamically adjusted.

```js
export let requestHostCallback;       
export let cancelHostCallback;
export let requestHostTimeout;
export let cancelHostTimeout;
export let shouldYieldToHost;
export let requestPaint;
export let getCurrentTime;
export let forceFrameRate;
```

下面是文件的代码（针对场景改过一些）

```js
if (
  // If Scheduler runs in a non-DOM environment, it falls back to a naive
  // implementation using setTimeout.
  typeof window === 'undefined' ||
  // Check if MessageChannel is supported, too.
  typeof MessageChannel !== 'function'
) {
  // 用setTimeout等模拟
} else {
  // Capture local references to native APIs, in case a polyfill overrides them.
  const performance = window.performance;
  const Date = window.Date;
  const setTimeout = window.setTimeout;
  const clearTimeout = window.clearTimeout;
  const requestAnimationFrame = window.requestAnimationFrame;
  const cancelAnimationFrame = window.cancelAnimationFrame;
  const requestIdleCallback = window.requestIdleCallback;

  // false
  const requestIdleCallbackBeforeFirstFrame = false  
    
  // performance.now() 可以获取到当前的页面打开到现在时间戳
  getCurrentTime = () => performance.now()

  let isRAFLoopRunning = false;
  let isMessageLoopRunning = false;
  let scheduledHostCallback = null;
  let rAFTimeoutID = -1;
  let taskTimeoutID = -1;

  // 这里先只考虑30帧，这样一帧的长度约等于33.33ms
  let frameLength = 33.33; 

  let prevRAFTime = -1;
  let prevRAFInterval = -1;
  let frameDeadline = 0;

  // 锁帧的开关
  let fpsLocked = false;

  // TODO: Make this configurable
  // TODO: Adjust this based on priority?
  let maxFrameLength = 300;
  let needsPaint = false;

  if (
    enableIsInputPending && // false
    navigator !== undefined &&
    navigator.scheduling !== undefined &&
    navigator.scheduling.isInputPending !== undefined
  ) {
    // ... 
  } else {
    // `isInputPending` is not available. Since we have no way of knowing if
    // there's pending input, always yield at the end of the frame.
    // 第一个重要的函数 shouldYieldToHost
    // 用来判断，当前是不是已经到了当前帧的末尾
    shouldYieldToHost = function () {
      return getCurrentTime() >= frameDeadline;
    };

    // Since we yield every frame regardless, `requestPaint` has no effect.
    requestPaint = function () { };
  }

  // 关于锁帧的函数
  // frameLength以及frameDeadline都是又这个来决定 
  forceFrameRate = function (fps) {
    if (fps < 0 || fps > 125) {
      console.error(
        'forceFrameRate takes a positive int between 0 and 125, ' +
        'forcing framerates higher than 125 fps is not unsupported',
      );
      return;
    }
    if (fps > 0) {
      frameLength = Math.floor(1000 / fps);
      fpsLocked = true;
    } else {
      // reset the framerate
      frameLength = 33.33;
      fpsLocked = false;
    }
  };

  // 第二个重要函数
  // 由port.onmessage = performWorkUntilDeadline触发
  // 在浏览器paint流程结束后触发
  const performWorkUntilDeadline = () => {
    // postMessage方案 or rAF方案  的标志
    // 后续pm已经替代了rAF
    if (enableMessageLoopImplementation) {
      if (scheduledHostCallback !== null) {
        const currentTime = getCurrentTime();
        // Yield after `frameLength` ms, regardless of where we are in the vsync
        // cycle. This means there's always time remaining at the beginning of
        // the message event.
        frameDeadline = currentTime + frameLength;
        const hasTimeRemaining = true;
        try {
          const hasMoreWork = scheduledHostCallback(
            hasTimeRemaining,
            currentTime,
          );
          if (!hasMoreWork) {
            isMessageLoopRunning = false;
            scheduledHostCallback = null;
          } else {
            // If there's more work, schedule the next message event at the end
            // of the preceding one.
            port.postMessage(null);
          }
        } catch (error) {
          // If a scheduler task throws, exit the current browser task so the
          // error can be observed.
          port.postMessage(null);
          throw error;
        }
      }
      // Yielding to the browser will give it a chance to paint, so we can
      // reset this.
      needsPaint = false;
    } else {
      // 在requestHostCallback流程里会赋值成callback
      if (scheduledHostCallback !== null) {
        // 通过performance.now()获取时间戳
        const currentTime = getCurrentTime();
        // 当前帧的剩余时间
        const hasTimeRemaining = frameDeadline - currentTime > 0;
        try {
          // 带着剩余时间和当前时间直接触发
          // 
          const hasMoreWork = scheduledHostCallback(
            hasTimeRemaining,
            currentTime,
          );
          // 没有剩余时间的时候直接把任务回调清空
          if (!hasMoreWork) {
            scheduledHostCallback = null;
          }
        } catch (error) {
          // If a scheduler task throws, exit the current browser task so the
          // error can be observed, and post a new task as soon as possible
          // so we can continue where we left off.
          // 如果任务抛出错误，这里会触发postMessage，
          // 而postMessage会触发新的performWorkUntilDeadline
          // 然后把错误继续抛出给外层捕获
          port.postMessage(null);
          throw error;
        }
      }
      // Yielding to the browser will give it a chance to paint, so we can
      // reset this.
      // 用来告诉浏览器主线程，应该进行paint的流程了
      // 这个是在计算过frameDeadline以后的情况
      // 如果不用这个标识来触发浏览器paint，有可能造成丢帧
      needsPaint = false;
    }
  };

  const channel = new MessageChannel();
  const port = channel.port2;
  // 所有的post.postMessage都会触发performWorkUntilDeadline
  // 然后判断有没有任务需要处理
  channel.port1.onmessage = performWorkUntilDeadline;


  // onAnimationFrame是requestAnimationFrame里的回调函数
  // 这里的rAFTime是requestAnimationFrame开始执行回调的时候的时间
  // 与performance.now()的返回值相同
  const onAnimationFrame = rAFTime => {
    if (scheduledHostCallback === null) {
      // No scheduled work. Exit.
      // 没有任务回调，直接置空返回
      prevRAFTime = -1;
      prevRAFInterval = -1;
      isRAFLoopRunning = false;
      return;
    }

    // Eagerly schedule the next animation callback at the beginning of the
    // frame. If the scheduler queue is not empty at the end of the frame, it
    // will continue flushing inside that callback. If the queue *is* empty,
    // then it will exit immediately. Posting the callback at the start of the
    // frame ensures it's fired within the earliest possible frame. If we
    // waited until the end of the frame to post the callback, we risk the
    // browser skipping a frame and not firing the callback until the frame
    // after that.
    // 表示requestAnimationFrame循环进行中
    // 再次调用requestAnimationFrame
    isRAFLoopRunning = true;
    requestAnimationFrame(nextRAFTime => {
      clearTimeout(rAFTimeoutID);
      onAnimationFrame(nextRAFTime);
    });

    // requestAnimationFrame is throttled when the tab is backgrounded. We
    // don't want to stop working entirely. So we'll fallback to a timeout loop.
    // TODO: Need a better heuristic for backgrounded work.
    const onTimeout = () => {
      // 把frameDeadline时间缩短，然后再调用performWorkUntilDeadline
      // todo？ 为什么要压短，凭什么除以2
      frameDeadline = getCurrentTime() + frameLength / 2;
      performWorkUntilDeadline();
      // 以3片的长度为1个单位
      rAFTimeoutID = setTimeout(onTimeout, frameLength * 3);
    };
    rAFTimeoutID = setTimeout(onTimeout, frameLength * 3);

    if (
      prevRAFTime !== -1 &&
      // Make sure this rAF time is different from the previous one. This check
      // could fail if two rAFs fire in the same frame.
      // 注释说的是确保当前帧的raf时间和上一个raf时间不同
      rAFTime - prevRAFTime > 0.1
    ) {
      // raf帧间距
      const rAFInterval = rAFTime - prevRAFTime;
      if (!fpsLocked && prevRAFInterval !== -1) {
        // We've observed two consecutive frame intervals. We'll use this to
        // dynamically adjust the frame rate.
        //
        // If one frame goes long, then the next one can be short to catch up.
        // If two frames are short in a row, then that's an indication that we
        // actually have a higher frame rate than what we're currently
        // optimizing. For example, if we're running on 120hz display or 90hz VR
        // display. Take the max of the two in case one of them was an anomaly
        // due to missed frame deadlines.

        // 这里是对帧率优化
        // 当单位帧的长度更短时，代表fps更高
        // 所以会动态把帧的长度变短
        if (rAFInterval < frameLength && prevRAFInterval < frameLength) {
          frameLength =
            rAFInterval < prevRAFInterval ? prevRAFInterval : rAFInterval;
          if (frameLength < 8.33) {
            // Defensive coding. We don't support higher frame rates than 120hz.
            // If the calculated frame length gets lower than 8, it is probably
            // a bug.
            frameLength = 8.33;
          }
        }
      }
      prevRAFInterval = rAFInterval;
    }
    prevRAFTime = rAFTime;
    // 可以说react的帧长度计算是从当前的raf开始，到下一个raf开始前结束
    // 不是纯粹的以浏览器的分片(frame start)为单位
    frameDeadline = rAFTime + frameLength;

    // We use the postMessage trick to defer idle work until after the repaint.
    // 在paint之后出发另一个scheduleCallback
    port.postMessage(null);
  };

  // callback触发器
  // 我们只需要主要在意postMessage或者rAF触发
  // 剩下的requestIdleCallback和setTimout都是处理某些情况下未触发第一次rAF循环
  // 也许这也是后面react抛弃rAF循环，采用postMessage的原因之一吧
  requestHostCallback = function (callback) {
    // 赋值到管理任务回调的全局变量
    scheduledHostCallback = callback;
    // 这个变量表示，调度的任务触发 是用postMessage，还是用raf
    // 在后续的版本里raf的方法被删除了
    if (enableMessageLoopImplementation) {
      if (!isMessageLoopRunning) {
        // postMessage循环的标志
        isMessageLoopRunning = true;
        port.postMessage(null);
      }
    } else {
      if (!isRAFLoopRunning) {
        // Start a rAF loop.
        isRAFLoopRunning = true;
        // 使用rAF开始任务循环
        requestAnimationFrame(rAFTime => {
          if (requestIdleCallbackBeforeFirstFrame) {
            cancelIdleCallback(idleCallbackID);
          }
          if (requestTimerEventBeforeFirstFrame) {
            clearTimeout(idleTimeoutID);
          }
          // 动态算帧的一个
          onAnimationFrame(rAFTime);
        });

        // If we just missed the last vsync, the next rAF might not happen for
        // another frame. To claim as much idle time as possible, post a
        // callback with `requestIdleCallback`, which should fire if there's
        // idle time left in the frame.
        //
        // This should only be an issue for the first rAF in the loop;
        // subsequent rAFs are scheduled at the beginning of the
        // preceding frame.

        // 在rAF循环的伊始，也可能会错过第一个rAF时机，以此错过整个rAF循环
        // 所以这里抓住当前帧的另一个可以处理任务的时机 - requestIdleCallback
        // 来触发rAF循环
        let idleCallbackID;
        if (requestIdleCallbackBeforeFirstFrame) {
          idleCallbackID = requestIdleCallback(
            function onIdleCallbackBeforeFirstFrame() {
              if (requestTimerEventBeforeFirstFrame) {
                clearTimeout(idleTimeoutID);
              }
              frameDeadline = getCurrentTime() + frameLength;
              performWorkUntilDeadline();
            },
          );
        }
        // Alternate strategy to address the same problem. Scheduler a timer
        // with no delay. If this fires before the rAF, that likely indicates
        // that there's idle time before the next vsync. This isn't always the
        // case, but we'll be aggressive and assume it is, as a trade off to
        // prevent idle periods.

        // 同样 用setTimeout来意外情况处理rAF
        let idleTimeoutID;
        if (requestTimerEventBeforeFirstFrame) {
          idleTimeoutID = setTimeout(function onTimerEventBeforeFirstFrame() {
            if (requestIdleCallbackBeforeFirstFrame) {
              cancelIdleCallback(idleCallbackID);
            }
            frameDeadline = getCurrentTime() + frameLength;
            performWorkUntilDeadline();
          }, 0);
        }
      }
    }
  };

  // 把任务回调置空
  cancelHostCallback = function () {
    scheduledHostCallback = null;
  };

  // 使用setTimout触发一次callback， 不是循环
  requestHostTimeout = function (callback, ms) {
    taskTimeoutID = setTimeout(() => {
      callback(getCurrentTime());
    }, ms);
  };

  // 取消上面的任务触发
  cancelHostTimeout = function () {
    clearTimeout(taskTimeoutID);
    taskTimeoutID = -1;
  };
}

```