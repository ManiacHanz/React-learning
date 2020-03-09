
从这一节开始，阅读关于schedule调度的部分，这一节开始涉及到react核心原理。
不关键的部分的话，我还是想先从简单原则下手，即初始化走了哪个分支就从哪个分支开始

```js
function scheduleRootUpdate(){

    // .....  有关update的操作
    // 这里的current是rootFiber 而expirationTime是Sync
    scheduleWork(current, expirationTime);
    return expirationTime
}
```

先了解
`executionContext`是一个全局变量，用来记录执行类型的上下文，总是在下面几个状态值中改变，初始化为`NoContext`
同时也可以通过位运算，来判断当前的执行状态

```js
const NoContext = /*                    */ 0b000000;
const BatchedContext = /*               */ 0b000001;
const EventContext = /*                 */ 0b000010;
const DiscreteEventContext = /*         */ 0b000100;
const LegacyUnbatchedContext = /*       */ 0b001000;     // 代表在unbatchedUpdates中
const RenderContext = /*                */ 0b010000;		 // 代表在render阶段
const CommitContext = /*                */ 0b100000;		 // 代表在commit阶段

// 比如说通过计算判断是否在unbatchedContext
(executionContext & LegacyUnbatchedContext) !== NoContext
// 比如说是否已经在render阶段
(executionContext & (RenderContext | CommitContext)) === NoContext
```

```js
// react-reconciler/src/ReactFiberWorkLoop


export function scheduleUpdateOnFiber(
  fiber: Fiber,
  expirationTime: ExpirationTime,
) {
  /**
    2个警告函数
    简单的说，这个是检查是否在componentWillUpdate或者componentDidUpdate使用setState
    或者说在useEffect中使用setState并且没有设置第二个参数[]
    导致的无线循环更新
    用的全局变量nestedUpdateCount来统计

    后者 是检测getChildContext 和 render中使用setState造成的栈溢出警告
   */
  checkForNestedUpdates();

  warnAboutInvalidUpdatesOnClassComponentsInDEV(fiber);

  /**
		这个函数的作用主要是从fiber.stateNode开始，持续更新expirationTime以及alternate.expirationTime优先级更高

    这个root是把expirationTime更新到了fiber.expirationTime，取了优先级更高（值更大）
    并且也可能是把当前的childExpiration更新并且一步步return出来的root

    但 应该不是所有的node，因为没有在child时候去找child的siblings，而是直接return
    相当于一头向上直到root

		当从ReactDOM.render进来相当于更新rootFiber.expirationTime以及rootFiber.alternate.expirationTime
   */
  const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);

  root.pingTime = NoWork;

  /**
		有一个全局的interruptedBy
    当更新流程优先级被打断的时候
		同时和dev环境有关
    好像是用来给react开发工具记录被哪个节点打断了异步任务
		略过
   */
  checkForInterruption(fiber, expirationTime);
  // 性能和报错有关
  recordScheduleUpdate();

  // TODO: computeExpirationForFiber also reads the priority. Pass the
  // priority as an argument to that function and this one.
  /**
	 * getCurrentPriorityLevel 是Schedule一个方法
	 * 作用是取得那个文件中一个表示权重值的全局变量currentPriorityLevel
	 * 该变量会在不同任务是修改，任务优先级越高权重值越大
   * 大概从高到低分了5个等级 从1-5数字越大代表优先级越低
   * 从ReactDom.render这里进来返回的是97
	 * 暂时不知道是哪个值过来的
   */
  const priorityLevel = getCurrentPriorityLevel();
	// 记得expirationTime进来的值为sync么
  if (expirationTime === Sync) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
			// 走进这里，如果不懂这个判断条件可以见上面
      // Register pending interactions on the root to avoid losing traced interaction data.
      // 记录interactBy  先不管
      schedulePendingInteractions(root, expirationTime);

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      // 表示render应该立即执行，但是更新应该推迟到批量更新之后
			// 于是要进入renderRoot 看看这个函数，已经返回的函数执行都是啥
      let callback = renderRoot(root, Sync, true);
      while (callback !== null) {
        callback = callback(true);
      }
    } else {
      scheduleCallbackForRoot(root, ImmediatePriority, Sync);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, wnless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of sync mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    scheduleCallbackForRoot(root, priorityLevel, expirationTime);
  }
  
  // reactDOM.render不会走到下面的条件里，
  // 至于何时走进来 todo
  if (
    (executionContext & DiscreteEventContext) !== NoContext &&
    // Only updates at user-blocking priority or greater are considered
    // discrete, even inside a discrete event.
    (priorityLevel === UserBlockingPriority ||
      priorityLevel === ImmediatePriority)
  ) {
    // This is the result of a discrete event. Track the lowest priority
    // discrete update per root so we can flush them early, if needed.
    if (rootsWithPendingDiscreteUpdates === null) {
      rootsWithPendingDiscreteUpdates = new Map([[root, expirationTime]]);
    } else {
      const lastDiscreteTime = rootsWithPendingDiscreteUpdates.get(root);
      if (lastDiscreteTime === undefined || lastDiscreteTime > expirationTime) {
        rootsWithPendingDiscreteUpdates.set(root, expirationTime);
      }
    }
  }
}
export const scheduleWork = scheduleUpdateOnFiber;



function markUpdateTimeFromFiberToRoot(fiber, expirationTime) {
  // Update the source fiber's expiration time
  // 把前面通过计算得来的expirationTime和fiber.expirationTime相比较，取权重更优先的
  if (fiber.expirationTime < expirationTime) {
    fiber.expirationTime = expirationTime;
  }
  // 有workInProgress的时候 也同理更新更优先的权重expirationTime
  let alternate = fiber.alternate;
  if (alternate !== null && alternate.expirationTime < expirationTime) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  /** 一遍遍历找到root节点，并且更新child的expirationTime为更优先（大）值 */
  let node = fiber.return;
  let root = null;
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode;
  } else {
    while (node !== null) {
      /**
				既要更新子节点的current的expirationTime，也要更新子节点workInProgress的expirationTime
				node.childExpirationTime已经是子节点中的最大值？@TODO 待验证
				注：但是这里没有找child的siblings，而是直接找的return
      */
      alternate = node.alternate;
      if (node.childExpirationTime < expirationTime) {
        node.childExpirationTime = expirationTime;
        if (
          alternate !== null &&
          alternate.childExpirationTime < expirationTime
        ) {
          alternate.childExpirationTime = expirationTime;
        }
      } else if (
        alternate !== null &&
        alternate.childExpirationTime < expirationTime
      ) {
        alternate.childExpirationTime = expirationTime;
      }
      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode;
        break;
      }
			// 更新了node以后并没有更新node.sibling
      node = node.return;
    }
  }

  if (root !== null) {
    // Update the first and last pending expiration times in this root
    /**
      更新root上的first 和last pending time。同样是取较大值
      ReactDOM.render 初始化的时候lastPendingTime和firstPengdingTime都是NoWork 也就是0
      所以这里都会更新成 Sync
    */
    const firstPendingTime = root.firstPendingTime;
    if (expirationTime > firstPendingTime) {
      root.firstPendingTime = expirationTime;
    }
    const lastPendingTime = root.lastPendingTime;
    if (lastPendingTime === NoWork || expirationTime < lastPendingTime) {
      root.lastPendingTime = expirationTime;
    }
  }

  return root;
}
```

`renderRoot`是一个相当长的方法，我们下章见