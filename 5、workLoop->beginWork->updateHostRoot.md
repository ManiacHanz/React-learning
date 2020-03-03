
接上一章，从`renderRoot`进来，开始执行render阶段的函数以后，里面最核心的一步就是`workLoop`工作循环，这一节主要就要详解这个`workLoop`的作用

可以看到`workLoop`或者`workLoopSync`都是执行`performUnitOfWork`, 按`workInProgress`的单元来执行，从命名可以感觉出来是个循环或者递归执行的过程。

`performUnitOfWork`看起来很多，梳理一下首先主要是执行了`beginWork`，然后把返回值执行了`completeUnitOfWork`。这两个函数都是很重要的流程，我们一个一个来分析

`beginWork`根据`Fiber`上的一些属性 `memoizedProps`, `pendingProps` 来处理新收到的props,这里和`context`以及`props`更新有关，最终根据`fiber.tag` 按照`component`的类型进行分别处理。这里是进入到`updateHostRoot`里

在`updateHostRoot`中，主要是对`updateQueue`整个列表进行处理，从`firstUpdate`开始，通过`next`往下进行一次遍历。最后完成对`updateQueue`以及`workInProgress`的操作。一个是赋值`workInProgress`的`expirationTime`和`memorizedState`，另一个是赋值`updateQueue`上的`baseState`,`firstUpdate`, `lastUpdate`等。（PS: 在`updateQueue`里还有一个链表叫做`capturedUpdate`，主要是用来处理`FiberThrow`的，可以先忽略这部分功能）。

最后调用调和子节点的方法`reconcileChildren`。这个方法就放在下一页看



```js
// react-reconciler/src/ReactFiberWorkLoop.js

// 在同步 或者 Scheduler 判定不用yield的情况下
// 执行performUnitOfWork(workInProgress)
// 这里的workInProgress是全局变量的workInProgress
// 在renderRoot中通过 prepareFreshStack 生成了一个
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function workLoop() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;

  // 和开发模式的devtools有关系
  startWorkTimer(unitOfWork);
  setCurrentDebugFiberInDEV(unitOfWork);

  // profiler先略过，直接理解成 进入else分支 next = beginWork()
  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, renderExpirationTime);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, renderExpirationTime);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(unitOfWork);
  }

  ReactCurrentOwner.current = null;
  return next;
}

```




```js
// react-reconciler/src/ReactFiberBeginWork.js
/*
  参数
  current和workInProgress互为alternate， current是rootFiber
  renderExpirationTime代表的就是传进来的expirationTime
  这里是前面通过prepareFreshStack重置后的，这里是Sync
*/
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  const updateExpirationTime = workInProgress.expirationTime;
  
  if (current !== null) {
    // 这两个属性此时为null
    // memoizedProps： 上一次渲染完成之后的props
    // pendingProps: 新的变动带来的新的props
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      // null !== null   true
      // hasLegacyContextChanged() 来自ReactFiberContext.js 返回boolean ，大概是用来控制 老版本 context有关。这里返回的false
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      // 
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
      // 这里 updateExpirationTime 是 workInProgress.expirationTime
      // renderExpiration 是 由于是renderRoot, 这里在prepareStack的时候被 定义成Sync
      // 而updateExpirationTime这里也是Sync.  所以这里是相等
    } else if (updateExpirationTime < renderExpirationTime) {
      didReceiveUpdate = false;
      // This fiber does not have any pending work. Bailout without entering
      // the begin phase. There's still some bookkeeping we that needs to be done
      // in this optimized path, mostly pushing stuff onto the stack.
      // 这下面还会根据组件的类型执行一些必要操作
      // renderRoot直接跳过
    }
  } else {
    didReceiveUpdate = false;
  }

  // Before entering the begin phase, clear the expiration time.
  // 清空expirationTime
  workInProgress.expirationTime = NoWork;
  // 这里就看出 tag的作用，根据tag的标志 分成classComponent functionComponent HostRoot Hosttext等等
  // 来分开进行处理，此时我们只看 HostRoot
  switch (workInProgress.tag) {
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderExpirationTime);
}
```

所以说`beginWork`只是一个准备工作，把`FiberNode`分类型，做准备工作等等。然后执行开始工作的函数 `updateHostRoot`

```js
// react-reconciler/src/ReactFiberBeginWork.js

function pushHostRootContext(workInProgress) {
  const root = (workInProgress.stateNode: FiberRoot);
  if (root.pendingContext) {
    pushTopLevelContextObject(
      workInProgress,
      root.pendingContext,
      root.pendingContext !== root.context,
    );
  } else if (root.context) {
    // Should always be set
    pushTopLevelContextObject(workInProgress, root.context, false);
  }
  pushHostContainer(workInProgress, root.containerInfo);
}


function updateHostRoot(current, workInProgress, renderExpirationTime) {
  pushHostRootContext(workInProgress);
  // updateQueue 已经被初始化了 ，有了一些基本的属性
  const updateQueue = workInProgress.updateQueue;
  // null , null, null
  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState !== null ? prevState.element : null;
  // 这个函数是遍历updateQueue链表 对updateQueue的属性进行操作 
  // 会遍历update的链表，以及capturedUpdate的链表  从firstUpdate，跟着next往下走  具体有
  // 遍历每个update上的expirationTime，找出优先级最高的，挂在 workInProgress上，标记成workInProgress的expirationTime
  // 遍历每个update，找到update或者capturedUpdate里优先级最高的update，尤其是 >= renderExpirationTime的权重的，
  //    --把workInProgress的memoizedState赋值成权重最高的update的baseState,这里的值其实与update的payload有关，如果是function就取执行结果
  //    --如果不是就取他自己，然后和之前prevState合并形成新的state
  // 根据update里的expirationTime和参数里的renderExpirationTime作比较，然后把queue上的baseState和firstUpdate赋值。
  // 这里处理完后，queue上的update链表会被清空，表示没有需要进行更新的操作
  // 而baseState是之前firstUpdate.payload里面取出来的element对象
  processUpdateQueue(
    workInProgress,
    updateQueue,
    nextProps,
    null,
    renderExpirationTime,
  );
  // memoizedState是在上面的processUpdateQueue 里处理的
  // 也就是拿到了最优处理的update。
  // 具体的可以看下面的getStateFromUpdate
  // 在update.expirationTime >= renderExpirationTime的时候，表示应该着重处理了。
  // 是拿的这个update.payload上的对象，和queue本来的memoizedState做合并，然后返回出来的baseState
  // 从reactDom.render进来这里只是firstUpdate.payload的一个element对象 -- 见图
  const nextState = workInProgress.memoizedState;
  // Caution: React DevTools currently depends on this property
  // being called "element".
  const nextChildren = nextState.element;
  // prevChildren此时===null
  // 因为之前的baseState是没有element的 是null.因为我们还是第一次进来
  if (nextChildren === prevChildren) { 
  }
  const root: FiberRoot = workInProgress.stateNode;
  if (
    // current是 FiberRoot
    (current === null || current.child === null) &&
    root.hydrate &&
    enterHydrationState(workInProgress)
  ) {
    
  } else {
    // 接下来就是调和子节点的流程了
    reconcileChildren(
      current,
      workInProgress,
      nextChildren,
      renderExpirationTime,
    );
    // 和调节一些全局变量有关，用于ReactFiberHydrationContext里的一些方法有关
    // 存一些正在处理的fiber对象和将要处理的fiber对象，这里是把这些都重置为null的方法，暂时可以不用看
    resetHydrationState();
  }
  return workInProgress.child;
}
```


------------------------

```js
// react-reconciler/src/ReactUpdateQueue.js
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  queue: UpdateQueue<State>,
  props: any,
  instance: any,
  renderExpirationTime: ExpirationTime,
): void {
  hasForceUpdate = false;
  // 确保是克隆不会影响原queue
  queue = ensureWorkInProgressQueueIsAClone(workInProgress, queue);

  // These values may change as we process the queue.
  let newBaseState = queue.baseState;
  let newFirstUpdate = null;
  let newExpirationTime = NoWork;

  // Iterate through the list of updates to compute the result.
  let update = queue.firstUpdate;
  let resultState = newBaseState;
  while (update !== null) {
    const updateExpirationTime = update.expirationTime;
    if (updateExpirationTime < renderExpirationTime) {
      // This update does not have sufficient priority. Skip it.
      if (newFirstUpdate === null) {
        // This is the first skipped update. It will be the first update in
        // the new list.
        newFirstUpdate = update;
        // Since this is the first update that was skipped, the current result
        // is the new base state.
        newBaseState = resultState;
      }
      // Since this update will remain in the list, update the remaining
      // expiration time.
      if (newExpirationTime < updateExpirationTime) {
        newExpirationTime = updateExpirationTime;
      }
    } else {
      // This update does have sufficient priority.


      // Process it and compute a new result.
      resultState = getStateFromUpdate(
        workInProgress,
        queue,
        update,
        resultState,
        props,
        instance,
      );
      const callback = update.callback;
      if (callback !== null) {
        workInProgress.effectTag |= Callback;
        // Set this to null, in case it was mutated during an aborted render.
        update.nextEffect = null;
        if (queue.lastEffect === null) {
          queue.firstEffect = queue.lastEffect = update;
        } else {
          queue.lastEffect.nextEffect = update;
          queue.lastEffect = update;
        }
      }
    }
    // Continue to the next update.
    update = update.next;
  }
  
  // capturedUpdate 代表和react错误边界有关，可以不用深入研究
  // Separately, iterate though the list of captured updates.
  let newFirstCapturedUpdate = null;
  update = queue.firstCapturedUpdate;
  while (update !== null) {
    const updateExpirationTime = update.expirationTime;
    if (updateExpirationTime < renderExpirationTime) {
      // This update does not have sufficient priority. Skip it.
      if (newFirstCapturedUpdate === null) {
        // This is the first skipped captured update. It will be the first
        // update in the new list.
        newFirstCapturedUpdate = update;
        // If this is the first update that was skipped, the current result is
        // the new base state.
        if (newFirstUpdate === null) {
          newBaseState = resultState;
        }
      }
      // Since this update will remain in the list, update the remaining
      // expiration time.
      if (newExpirationTime < updateExpirationTime) {
        newExpirationTime = updateExpirationTime;
      }
    } else {
      // This update does have sufficient priority. Process it and compute
      // a new result.
      resultState = getStateFromUpdate(
        workInProgress,
        queue,
        update,
        resultState,
        props,
        instance,
      );
      const callback = update.callback;
      if (callback !== null) {
        workInProgress.effectTag |= Callback;
        // Set this to null, in case it was mutated during an aborted render.
        update.nextEffect = null;
        if (queue.lastCapturedEffect === null) {
          queue.firstCapturedEffect = queue.lastCapturedEffect = update;
        } else {
          queue.lastCapturedEffect.nextEffect = update;
          queue.lastCapturedEffect = update;
        }
      }
    }
    update = update.next;
  }
  // 此时链表只有一个update first === last 且expirationTime === Sync
  // 所以把lastUpdate清掉没有影响
  if (newFirstUpdate === null) {
    queue.lastUpdate = null;
  }
  if (newFirstCapturedUpdate === null) {
    queue.lastCapturedUpdate = null;
  } else {
    workInProgress.effectTag |= Callback;
  }
  if (newFirstUpdate === null && newFirstCapturedUpdate === null) {
    // We processed every update, without skipping. That means the new base
    // state is the same as the result state.
    newBaseState = resultState;
  }
  // newBaseState此时是原firstUpdate上的payload的element对象
  queue.baseState = newBaseState;
  // 而后两个置位空？？
  queue.firstUpdate = newFirstUpdate;
  queue.firstCapturedUpdate = newFirstCapturedUpdate;

  // Set the remaining expiration time to be whatever is remaining in the queue.
  // This should be fine because the only two other things that contribute to
  // expiration time are props and context. We're already in the middle of the
  // begin phase by the time we start processing the queue, so we've already
  // dealt with the props. Context in components that specify
  // shouldComponentUpdate is tricky; but we'll have to account for
  // that regardless.
  workInProgress.expirationTime = newExpirationTime;
  workInProgress.memoizedState = resultState;
}

/// 

function getStateFromUpdate<State>(
  workInProgress: Fiber,
  queue: UpdateQueue<State>,
  update: Update<State>,
  prevState: State,   
  nextProps: any,     // null
  instance: any,      // undefined
): any {
  switch (update.tag) {   // 这里等于 updateState   0
    case ReplaceState: {
      const payload = update.payload;
      if (typeof payload === 'function') {
        const nextState = payload.call(instance, prevState, nextProps);
        return nextState;
      }
      // State object
      return payload;
    }
    case CaptureUpdate: {
      workInProgress.effectTag =
        (workInProgress.effectTag & ~ShouldCapture) | DidCapture;
    }
    // Intentional fallthrough
    case UpdateState: {
      const payload = update.payload;
      let partialState;
      if (typeof payload === 'function') {
        partialState = payload.call(instance, prevState, nextProps);
      } else {
        // Partial state object
        partialState = payload;
      }
      if (partialState === null || partialState === undefined) {
        // Null and undefined are treated as no-ops.
        return prevState;
      }
      // Merge the partial state and the previous state.
      return Object.assign({}, prevState, partialState);
    }
    case ForceUpdate: {
      hasForceUpdate = true;
      return prevState;
    }
  }
  return prevState;
}


```