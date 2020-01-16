
从`scheduleWork`那个方法，触发最后的`renderRoot(root, Sync, true)`。这里的root是`fiberRoot`，要去取`current`属性才会是`rootFiber`

```js
function renderRoot(
  root: FiberRoot,
  expirationTime: ExpirationTime,
  isSync: boolean,
): SchedulerCallback | null {

  // 这里有一些条件判断代码，都是都不会走进来，有兴趣自己看
  // flushPassiveEffects(); 和副作用相关，这里暂时没有作用
  

  // If the root or expiration time have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  // 在ReactDOM.render的时候,workInProgressRoot还为null,这里不知道要重新看一下current和workInProgress的关系
  // workInProgressRoot会在prepareFreshStack里赋值
  // expirationTime 此时为Sync ，
  // 而renderExpirationTime此时为NoWork，唯一会变成expirationTime的时候就是在下面的prepareFreshStack方法里
  // 在commit阶段会重置为NoWork
  if (root !== workInProgressRoot || expirationTime !== renderExpirationTime) {
    // freshStack  大致是重置一些root上的属性，以及这个文件里的全局变量
    // 同时会创建或者修改workInProgress，使值同current的同步，或重置
    prepareFreshStack(root, expirationTime);
    // 将调度优先级高的interaction加入到interactions中
    // 用一个 Set类型呃变量 interactions，存储优先级比expirationTime更高的interaction
    // 相当于当前的操作被优先级更高的操作插队了
    // 
    startWorkOnPendingInteractions(root, expirationTime);
  } else if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
    // 和被推迟的低优先级的更新相关 ，先略
  }

  // If we have a work-in-progress fiber, it means there's still work to do
  // in this root.
  // workInProgress只有在初始化的时候是null,然后在commit阶段结束后为Null
  // 这里是刚刚被赋值成的createWorkInProgress，所以需要往里面去进行一系列操作
  if (workInProgress !== null) {
    // 把当前的executionContext记录下来，然后用|= 表示进入render状态
    // 在后面再把执行上下文复原
    const prevExecutionContext = executionContext;
    // 0b001000 | 0b010000
    executionContext |= RenderContext;
    // 这里 = null ，和hooks相关...略过
    

    // isSync = true  expirationTime === Sync
    if (isSync) {
      if (expirationTime !== Sync) {
        // An async update expired. There may be other expired updates on
        // this root. We should render all the expired work in a
        // single batch.
        // 表示有些异步的更新已经失效了，这里要重新计算时间权重，然后递归重新调用renderRoot
        // 并且不再传Sync的值进去了
        const currentTime = requestCurrentTime();
        if (currentTime < expirationTime) {
          // Restart at the current time.
          executionContext = prevExecutionContext;
          resetContextDependencies();
          ReactCurrentDispatcher.current = prevDispatcher;
          if (enableSchedulerTracing) {
            __interactionsRef.current = ((prevInteractions: any): Set<
              Interaction,
            >);
          }
          return renderRoot.bind(null, root, currentTime);
        }
      }
    } else {
      // Since we know we're in a React event, we can clear the current
      // event time. The next update will compute a new event time.
      // currentEventTime会在 requestCurrentTime()方法执行时更新，
      // 而requestCurrentTime（按照我们当前）的顺序来看是在updateContainer执行的
      // 并且已经给后续的流程用完了
      // 这里把它重置，是的下一轮的update过程，又可以获得一个新的时间权重
      currentEventTime = NoWork;
    }

    do {
      try {
        // workLoop  一个递归函数 
        if (isSync) {
          workLoopSync();
        } else {
          workLoop();
        }
        break;
      } catch (thrownValue) {
        // 如果有错误，有一系列的错误捕获，变量重置，以及执行completeUnitOfWork
        // completeUnitOfWork后续会涉及，大概就是这里捕获了错误，也需要继续把fiber对象处理
        // 继续执行调度流程
      }
    } while (true);
    // 在workLoop执行完以后，代表commit阶段表示完毕
    executionContext = prevExecutionContext;
    
    // dispatcher和hooks相关
    resetContextDependencies();
    ReactCurrentDispatcher.current = prevDispatcher;
    if (enableSchedulerTracing) {
      __interactionsRef.current = ((prevInteractions: any): Set<Interaction>);
    }
    // 这里的workInProgress 在workLoop时已经被改变，这里为null
    // 后面会仔细讲workLoop
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
  stopFinishedWorkLoopTimer();
  
  // 当前的commit阶段结束，把更新过后的workInProgress赋值给root.finishedWork上
  // 把expirationTime付给root.finishedExpirationTime来保存
  root.finishedWork = root.current.alternate;
  root.finishedExpirationTime = expirationTime;

  // 和root.firstBatch记录的东西相关，
  // 这里返回false，先不看
  const isLocked = resolveLocksOnRoot(root, expirationTime);
  if (isLocked) {
    // This root has a lock that prevents it from committing. Exit. If we begin
    // work on the root again, without any intervening updates, it will finish
    // without doing additional work.
    return null;
  }

  // Set this to null to indicate there's no in-progress render.
  // 重置workInProgressRoot
  workInProgressRoot = null;

  // workInProgressRootExitStatus
  // 全局变量，代表这次commit以什么方式结束
  // 在render 被suspend, didError的时候会被赋值，代表commit出了问题
  // 这里是正常的 RootCompleted
  switch (workInProgressRootExitStatus) {
    case RootIncomplete: {
    }
    case RootErrored: {
    }
    case RootSuspended: {
    }
    case RootSuspendedWithDelay: {
    }
    case RootCompleted: {
      // The work completed. Ready to commit.
      if (
        !isSync &&
        // do not delay if we're inside an act() scope
        !(
          __DEV__ &&
          flushSuspenseFallbacksInTests &&
          IsThisRendererActing.current
        ) &&
        workInProgressRootLatestProcessedExpirationTime !== Sync &&
        workInProgressRootCanSuspendUsingConfig !== null
      ) {
        // If we have exceeded the minimum loading delay, which probably
        // means we have shown a spinner already, we might have to suspend
        // a bit longer to ensure that the spinner is shown for enough time.
        const msUntilTimeout = computeMsUntilSuspenseLoadingDelay(
          workInProgressRootLatestProcessedExpirationTime,
          expirationTime,
          workInProgressRootCanSuspendUsingConfig,
        );
        if (msUntilTimeout > 10) {
          root.timeoutHandle = scheduleTimeout(
            commitRoot.bind(null, root),
            msUntilTimeout,
          );
          return null;
        }
      }
      return commitRoot.bind(null, root);
    }
    default: {
      invariant(false, 'Unknown root exit status.');
    }
  }
}
```

`prepareFreshStack`，表示要准备一个新的栈，也就是在有些被打断的时候要通过frest机制防止后续更新影响之前更新到一半的状态
*可以理解成全部初始化*
`createWorkInProgress`，创造或者修改一个`workInProgress`对象，方法很简单，可以大概浏览一下这里面的属性以及相应的注释，有个印象

```js
function prepareFreshStack(root, expirationTime) {
  root.finishedWork = null;
  root.finishedExpirationTime = NoWork;
  // root.timeoutHandle 初始化为noTimeout 就是-1
  // cancelTimeout和清理超时有关，涉及到调度部分的时间内容。先不管
  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }
  // workInProgress 是个全局变量，只有在第一次从ReactDOM.render进来以及完成了commit阶段的时候会设置成null
  // 用来表示需要调度的单元，后面会在performUnitOfWork等等的流程中看到这个变量
  // 这里的场景是上次被中断的更新流程，这里通过unwindInterruptedWork去从子到父的清理
  // 是根据component类型不同清理的不同，
  // 具体看unwindInterruptedWork。这里不是主流程暂时不细看
  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }
  // 把下面的全局变量全部初始化
  // 这些全局变量会在不同的调度过程中变成不同的值
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null, expirationTime);
  renderExpirationTime = expirationTime;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootLatestProcessedExpirationTime = Sync;
  workInProgressRootLatestSuspenseTimeout = Sync;
  workInProgressRootCanSuspendUsingConfig = null;
  workInProgressRootHasPendingPing = false;
}



// react-reconciler/src/ReactFiber.js

// This is used to create an alternate fiber to do work on.
export function createWorkInProgress(
  current: Fiber,
  pendingProps: any,
  expirationTime: ExpirationTime,
): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.

    // createFiber 是new 了一个 FiberNode
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode,
    );
    // 除了初始化的属性以外，之前current已经处理过的一些属性，此时要负载workInProgress上。下面都是老面孔了
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    // workInProgress和current 互为alternate
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;

    // We already have an alternate.
    // Reset the effect tag.
    
    // 要重置已有的workInProgress的effectTag
    // 同时effect的链表也要重置
    workInProgress.effectTag = NoEffect;

    // The effect list is no longer valid.
    workInProgress.nextEffect = null;
    workInProgress.firstEffect = null;
    workInProgress.lastEffect = null;

    if (enableProfilerTimer) {
      // We intentionally reset, rather than copy, actualDuration & actualStartTime.
      // This prevents time from endlessly accumulating in new commits.
      // This has the downside of resetting values for different priority renders,
      // But works for yielding (the common case) and should support resuming.
      workInProgress.actualDuration = 0;
      workInProgress.actualStartTime = -1;
    }
  }

  // 和current同步的属性
  workInProgress.childExpirationTime = current.childExpirationTime;
  workInProgress.expirationTime = current.expirationTime;

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          expirationTime: currentDependencies.expirationTime,
          firstContext: currentDependencies.firstContext,
          responders: currentDependencies.responders,
        };

  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  return workInProgress;
}
```