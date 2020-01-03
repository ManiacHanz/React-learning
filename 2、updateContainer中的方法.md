


获取`currentTime`的`requestCurrentTime`
这里通过断点，从`ReactDOM.render`方法进来会走进第一个分支，我们直接看第一个分支
*`currentEventTime`会在React Event调度的时候重新定义成`NoWork`，所以中间的直接返回currentEventTime应该是在非React Event内部，具体是是什么后面再看*

```js
// react-reconciler/src/ReactFiberWorkLoop.js

// const currentTime = requestCurrentTime();

/**
 * 获取currentTime
 * 通过第一次ReactDom.render进来的时候会执行第一个分支
 * 这是 executionContext = 0b001000
 * 同时RenderContext和CommitContext也都有值
 * 所以直接进msToExpirationTime()
 */
export function requestCurrentTime() {
  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    // We're inside React, so it's fine to read the actual time.
    return msToExpirationTime(now());
  }
  // We're not inside React, so we may be in the middle of a browser event.
  if (currentEventTime !== NoWork) {
    // Use the same start time for all updates until we enter React again.
    return currentEventTime;
  }
  // This is the first update since React yielded. Compute a new start time.
  currentEventTime = msToExpirationTime(now());
  return currentEventTime;
}
```

`msToExpirationTime(now())`
随着程序的进行，由于now()会越来越大，所以返回的值会原来越小，但是10ms以内的计算会得到同一个值

```js
// react-reconciler/src/ReactFiberExpirationTime.js

// MAX_SIGNED_31_BIT_INT 代表 2进制里最大的数  0b后面30位1
export const Sync = MAX_SIGNED_31_BIT_INT;
export const Batched = Sync - 1;

const UNIT_SIZE = 10;
/**
 * MAGIC_NUMBER_OFFSET 2的偏移避免和NoWork冲突
 */
const MAGIC_NUMBER_OFFSET = Batched - 1;

// 1 unit of expiration time represents 10ms.
export function msToExpirationTime(ms: number): ExpirationTime {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  /**
   * | 0 相当于取整
   * /10 相当于给个最小单位从1变成了10，这10以内的误差就不计算了？？？
   * 相当于 1000 - 100 / 10 和 1000 - 109 / 10 的结果其实是一样的
   */
  return MAGIC_NUMBER_OFFSET - ((ms / UNIT_SIZE) | 0);
}
```

在拿到`currentTime`以后就要去计算`expirationTime`了

```js
// const expirationTime = computeExpirationForFiber(
//     currentTime,     // 程序开始后相对最大的值
//     current,       // rootFiber
//     suspenseConfig,    // null
// );

// react-reconciler/src/ReactFiberWorkLoop.js

export function computeExpirationForFiber(
  currentTime: ExpirationTime,
  fiber: Fiber,
  suspenseConfig: null | SuspenseConfig,
): ExpirationTime {
  /** 
    mode = 0
    用来描述当前Fiber和他子树的`Bitfield`
    共存的模式表示这个子树是否默认是异步渲染的
   */
  const mode = fiber.mode;
  /**
   * 第一次进来的时候 mode 0 所以 与上 任何值肯定都是0  === NoMode
   * 所以直接把最大的数字返回回去 ，优先级最高
   */
  if ((mode & BatchedMode) === NoMode) {
    return Sync;
  }
  // 获取优先级 一共有5个等级
  // 再根据不同
  const priorityLevel = getCurrentPriorityLevel();
  if ((mode & ConcurrentMode) === NoMode) {
    return priorityLevel === ImmediatePriority ? Sync : Batched;
  }

  if ((executionContext & RenderContext) !== NoContext) {
    // Use whatever time we're already rendering
    return renderExpirationTime;
  }

  // 关于lazy和suspense， 先略过
  let expirationTime;
  if (suspenseConfig !== null) {
    // Compute an expiration time based on the Suspense timeout.
    expirationTime = computeSuspenseExpiration(
      currentTime,
      suspenseConfig.timeoutMs | 0 || LOW_PRIORITY_EXPIRATION,
    );
  } else {
    // Compute an expiration time based on the Scheduler priority.
    switch (priorityLevel) {
      case ImmediatePriority:
        expirationTime = Sync;
        break;
      case UserBlockingPriority:
        // TODO: Rename this to computeUserBlockingExpiration
        expirationTime = computeInteractiveExpiration(currentTime);
        break;
      case NormalPriority:
      case LowPriority: // TODO: Handle LowPriority
        // TODO: Rename this to... something better.
        expirationTime = computeAsyncExpiration(currentTime);
        break;
      case IdlePriority:
        expirationTime = Never;
        break;
      default:
        invariant(false, 'Expected a valid priority level');
    }
  }

  // If we're in the middle of rendering a tree, do not update at the same
  // expiration time that is already rendering.
  // TODO: We shouldn't have to do this if the update is on a different root.
  // Refactor computeExpirationForFiber + scheduleUpdate so we have access to
  // the root when we check for this condition.
  if (workInProgressRoot !== null && expirationTime === renderExpirationTime) {
    // This is a trick to move this update into a separate batch
    expirationTime -= 1;
  }

  return expirationTime;
}
```