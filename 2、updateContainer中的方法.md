


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

在拿到`currentTime`以后就要去计算`expirationTime`了。当然这里先涉及的是`computeExpirationForFiber`，就先看这里

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
   * 第一次进来的时候 mode 0 
   * 而BatchedMode = 0b0010
   * 所以&位运算算出来也是0 也就是=== NoMode
   * 所以直接把最大的数字返回回去 ，优先级最高
   */
  if ((mode & BatchedMode) === NoMode) {
    return Sync;
  }
  // 获取优先级 一共有5个等级
  // 再根据不同 priorityLevel以及suspenseConfig，
  // 判断应该用何种方法去计算expirationTime
  // 下面都属于异步的更新，暂时略过
  // .... 

  return expirationTime;
}
```

拿到值为`Sync`的`expirationTime`后，我们就可以来进入下一个方法了 `updateContainerAtExpirationTime`

```js
updateContainerAtExpirationTime(
  element,          //  <App />
  container,        // FiberRoot
  parentComponent,   // null
  expirationTime,    // Sync
  suspenseConfig,    // null or undefined
  callback,           // undefined
);
```

从这个方法开始，就要进入到react的核心阶段了

```js
// react-reconciler/src/ReactFiberReconciler.js

export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
  callback: ?Function,
) {
  // TODO: If this is a nested container, this won't be the root.
  const current = container.current;

  /** ReactDOM.render的时候  context = {} */
  const context = getContextForSubtree(parentComponent);
  /** ReactDOM.render的时候  container.context === null */
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }
  /**
   *  current 是rootFiber对象 element为ReactDOM.render()的第一个参数nodelist
   *  expirationTime 从render走进来的时候会是Sync最大值
   *  后两个为null
   */
  return scheduleRootUpdate(
    current,
    element,
    expirationTime,
    suspenseConfig,
    callback,
  );
}



function getContextForSubtree(
  parentComponent: ?React$Component<any, any>,
): Object {
  if (!parentComponent) {
    // emptyContextObject = {}
    return emptyContextObject;
  }
  // ...
}
```

在前面一系列的操作 -- 姑且叫初始化阶段 -- 之后，就要进入核心的`schedule`调度阶段。

在`scheduleRootUpdate`调度`root`的方法里，主要分为两步
* 创建`update`对象，并且挂到`rootFiber`对象上面，方便后续更新
* 执行`scheduleWork`，进入调度阶段

```js
// react-reconciler/src/ReactFiberReconciler.js

/**
 * 第一次进来
 * @param {*} current
 * @param {*} element
 * @param {*} expirationTime
 * @param {*} suspenseConfig
 * @param {*} callback
 */
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
  callback: ?Function,
) {
  /** createUpdate
   * 返回的一个update 对象
   * 里面比较重要的有 tag:0  expirationTime, suspenseConfig 以及next和nextEffect
   */
  const update = createUpdate(expirationTime, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  /** 给update对象挂上element */
  update.payload = {element};
  /**
    如果不用react-redux会在第三个参数传入状态变化的监听
    这里就会有callback
   ps：很少见过传第三个参数，只见过用redux的时候，所以这里姑且认为是undefined，怎么简单怎么来
   */
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  /** 在这个文件中这个变量没有被改变 ，这里为false */
  if (revertPassiveEffectsChange) {
    flushPassiveEffects();
  }
  /** 下面两个方法比较重要，也是核心，一个是把update加入队列，一个是任务调度
  两个的第一个参数都是rootFiber
   */
  /**
    enqueueUpdate
    首先去获取current也就是这个fiber的alternate，也就是workInProgress
    用queue1代表current.updateQueue用queue2代表alternate.updateQueue
    最终的目的是把fiber上的updateQueue这个属性，挂上处理后的update对象，以及在updateQueue上配置last以及next
    形成一个updateQueue链表闭环

    在ReactDOM.render进来的时候，这里其实只把current上的updateQueue形成了一个链表，
    把update赋值到了current.updateQueue.firstUpdate = current.updateQueue.lastUpdate = update;
   */
  enqueueUpdate(current, update);
  /**
    进入调度阶段，这里的expirationTime是前面通过计算得来的
    对于root而言 这里是Sync
   */
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

来看看 `update`相关的方法。
`createUpdate`，很简单，就是单纯的返回一个初始化对象，可以大概浏览一下上面有哪些属性，有个印象  

```js
// react-reconciler/src/ReactUpdateQueue.js

export function createUpdate(
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
): Update<*> {
  let update: Update<*> = {
    expirationTime,
    suspenseConfig,

    tag: UpdateState,
    payload: null,
    callback: null,

    next: null,
    nextEffect: null,
  };
  return update;
}


/**
 * 把update对象放入队列
 * @param {*} fiber fiber对象
 * @param {*} update createUpdate出来的update对象，并且挂在了update.payload
 */
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues are created lazily.
  /**
   * fiber在更新的过程中会有一个对应的fiber，挂载alternate上
   * 在代码中我们会看到current <==> workInProgress
   * 在渲染完成后，他们会交换位置，workInProgress完成的会变成current当前的
   * 而另外一个会更新后变成下一个 需要在变化中处理的fiber ？
   *
   * queue1用来记录current上面的updateQueue，queue2用来记录alternate上的updateQueue
   * 最终都会在queue1 queue2上挂载 memoizedState（这个具体是啥暂时@todo）
   * 并且通过appendUpdateToQueue 把queue上的 firstUpdate, lastUpdate, lastUpdate.next都配置上
   * 形成一个闭环链表
   */
  const alternate = fiber.alternate;
  let queue1;
  let queue2;

  if (alternate === null) {
    /**
      从ReactDOM.render走进来的时候，由于是第一次挂载，alternate === null
      且fiber.updateQueue和fiber.memoizedState都是null
    */
    // There's only one fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      /**
        此时相当于是一个初始化的queue1以及 fiber.updateQueue
       */
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // Only one fiber has an update queue. Clone to create a new one.
        /** queue1 === null && queue2 !== null 直接把queue2复制给queue2 */
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  /**
    下面的步骤通过appendUpdateToQueue即把update挂载到queue上的last和next
    形成一个闭环链表

    这里分了很多种情况并且在不用情况下写了不同处理，我们这里后期todo
  */
  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    /** 初始化的时候就会走到这里
      queue2 === null或者两个引用地址相同
      这里相当于把queue1的firstQueue和lastQueue都指向update
      也就是说这个链表只有这一项
    */
    appendUpdateToQueue(queue1, update);
  } else {
    // There are two queues. We need to append the update to both queues,
    // while accounting for the persistent structure of the list — we don't
    // want the same update to be added multiple times.
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}


```

