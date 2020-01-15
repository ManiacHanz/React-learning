
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

```js
// react-reconciler/src/ReactFiberWorkLoop


export function scheduleUpdateOnFiber(
  fiber: Fiber,
  expirationTime: ExpirationTime,
) {
  /**
    警告函数
    简单的说，这个是检查是否在componentWillUpdate或者componentDidUpdate使用setState
    或者说在useEffect中使用setState并且没有设置第二个参数[]
    导致的无线循环更新
    用的全局变量nestedUpdateCount来统计
   */
  checkForNestedUpdates();
  /**
    getChildContext 和 render中使用setState造成的栈溢出警告
   */
  warnAboutInvalidUpdatesOnClassComponentsInDEV(fiber);

  /**
    从ReactDOM.render进来传入的expirationTime为Sync
    这个root是把expirationTime更新到了fiber.expirationTime，取了优先级更高（值更大）
    并且也可能是把当前的childExpiration更新并且一步步return出来的root

    但 应该不是所有的node，因为没有在child时候去找child的siblings，而是直接return
    相当于一头向上知道root
   */
  const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);

  root.pingTime = NoWork;

  /**
    更改全局interruptedBy的值
    好像是用来给react开发工具记录被哪个节点打断了异步任务
   */
  checkForInterruption(fiber, expirationTime);
  /**
    记录hook相关的schedule, 主要是和性能有关
    放在后面看
   */
  recordScheduleUpdate();

  // TODO: computeExpirationForFiber also reads the priority. Pass the
  // priority as an argument to that function and this one.
  /**
   * 涉及到schedular中的一个过程，返回的是一个数字，用来表示优先级
   * 大概从高到低分了5个等级 从1-5数字越大代表优先级越低
   * 从ReactDom.render这里进来返回的是normalPriority，最后返回的是97
   */
  const priorityLevel = getCurrentPriorityLevel();

  if (expirationTime === Sync) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      /**
        executionContext = LegacyUnbatchedContext = 0b001000
        RenderContext = 0b010000   CommitContext = 0b100000
        会走到这个分支来
       */
      // Register pending interactions on the root to avoid losing traced interaction data.
      // 记录interactBy  先不管
      schedulePendingInteractions(root, expirationTime);

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      /**
        这里renderRoot开始就是循环遍历更新所以子节点的过程
        涉及到performUnitOfWork  beginWork等等
        这里复杂的放在后面需要单独解释
       */
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

```