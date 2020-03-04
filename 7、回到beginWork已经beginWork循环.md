
从上一章执行完调和子节点的fiber对象以后 会回到performUnitOfWork

如果说回到`performUnitOfWork`, 由于`next!==null`，我们就再通过return 往回看，看上一层

```js
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // 回忆一下这里的current就是current  unitOfwork其实是 workInProgress
  const current = unitOfWork.alternate;
  // ... 把其他代码先去掉了
  next = beginWork(current, unitOfWork, renderExpirationTime);
  // 这个和dev有关，先不看
  resetCurrentDebugFiberInDEV();

  // 把上一次延迟的属性放在memoizedProps里面，这样下一次就不必进行重复的更新
  // 当然接着之前的流程来看 这里是null 应该表示的没有被中断的更新或是怎么样  todo
  // ps: 全局搜 .pendingProps =    能发现他的赋值主要是 ReactBiberReconciler.js里的overrideProps里面
  // 而overrideProps是在injectIntoDevTools方法里被调用，而全局搜索这个方法也没找到明确的用途，这里先放一放 猜测和devtools有关系
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(unitOfWork);
  }
  // 全局变量  一般是用来在current上面保存workInProgress对象
  ReactCurrentOwner.current = null;
  return next;
}
```

往上一层看, 通过workInProgress来判断遍历，而这是判断条件里面的`workInProgress=performUnitOfWork(workInProgress);`这个返回起来的是workInProgress.child

```js
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
```

这里会循环遍历完整个workInProgress，然后在往回，这里会走到renderRoot 。接着在一系列判断流程
* 此时如果在work阶段有异常退出了，会重新进入renderRoot阶段
* 也有在commit阶段异常退出的，就会再进入commit阶段
* 正常情况 直接进入commitRoot阶段 (null, root)

我们在下一章开始讲解root阶段了

```js
  if(workInProgress !== null) {
    do {
      try {
        if (isSync) {
          workLoopSync();
        } else {
          workLoop();
        }
        break;
      }catch (thrownValue) {
        /// 略去catch处理
      }
    } while (true);
    
    // context以及全局变量 略

    // 这里在workLoop完成以后出来的的workInProgress肯定是null
    // 因为已经遍历完成
    // 如果此时不为null, 说明workInProgress仍未遍历完整颗树，需要记录一下expirationTime等座另外处理
    // 这里就会执行到仍然返回renderRoot函数，下次继续执行workLoop的循环，执行完了才会进入commit阶段
    if (workInProgress !== null) {
      // There's still work left over. Return a continuation.
      stopInterruptedWorkLoopTimer();
      if (expirationTime !== Sync) {
        startRequestCallbackTimer();
      }
      return renderRoot.bind(null, root, expirationTime);
    }
  }

  // We now have a consistent tree. The next step is either to commit it, or, if
  // something suspended, wait to commit it after a timeout.
  // 结束workLoop阶段  timer好像是只用于devtools或者是debugPerf之类的
  stopFinishedWorkLoopTimer();

  // 注意  此时的.current.alternate  和 workInProgress的区别
  // workInProgress是代指的workLoop循环下来的每一个fiber，通过找寻workInProgress.child不断的遍历完成的循环，此时已经=null
  // 而root.current.alternate 是更新完成的workInProgress，即一个完整的tree，只是还没有更新current
  // 此时先赋值给root.finishedWork来保存
  root.finishedWork = root.current.alternate;
  // 继续保存expirationTime
  root.finishedExpirationTime = expirationTime;

  // 看是否需要延迟commit阶段。主要是通过firstBatch上的_defer或者_expirationTime属性来决定，暂时不看
  const isLocked = resolveLocksOnRoot(root, expirationTime);
  if (isLocked) {
    // This root has a lock that prevents it from committing. Exit. If we begin
    // work on the root again, without any intervening updates, it will finish
    // without doing additional work.
    return null;
  }

  // Set this to null to indicate there's no in-progress render.
  // 这里在workLoop阶段之前  通过prepareFreshStack 赋值成了rootfiber，此时阶段结束了，用来设置成null
  workInProgressRoot = null;

  // 此时 = 4 
  switch (workInProgressRootExitStatus) {
    case RootIncomplete: {
      // 0
      invariant(false, 'Should have a work-in-progress.');
    }
    // Flow knows about invariant, so it complains if I add a break statement,
    // but eslint doesn't know about invariant, so it complains if I do.
    // eslint-disable-next-line no-fallthrough
    case RootErrored: {
      // 1 root错误，会根据错误判断是重新走renderRoot阶段 还是进入commit阶段，
      // 也就是在这两个阶段有标记是在哪个阶段中断的
      // An error was thrown. First check if there is lower priority work
      // scheduled on this root.
      const lastPendingTime = root.lastPendingTime;
      if (lastPendingTime < expirationTime) {
        // There's lower priority work. Before raising the error, try rendering
        // at the lower priority to see if it fixes it. Use a continuation to
        // maintain the existing priority and position in the queue.
        return renderRoot.bind(null, root, lastPendingTime);
      }
      if (!isSync) {
        // If we're rendering asynchronously, it's possible the error was
        // caused by tearing due to a mutation during an event. Try rendering
        // one more time without yiedling to events.
        prepareFreshStack(root, expirationTime);
        scheduleSyncCallback(renderRoot.bind(null, root, expirationTime));
        return null;
      }
      // If we're already rendering synchronously, commit the root in its
      // errored state.
      return commitRoot.bind(null, root);
    }
    case RootSuspended: {
      // 2  ...
      
    }
    case RootSuspendedWithDelay: {
      // 3 ... 
    }
    case RootCompleted: {
      // 4   表示work阶段结束，准备开始commit阶段
      // The work completed. Ready to commit.
      //  条件省略 不看 也就是说即将进入commitRoot阶段了
      return commitRoot.bind(null, root);
    }
    default: {
      invariant(false, 'Unknown root exit status.');
    }
  }

```