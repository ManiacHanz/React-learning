

```js
// react-reconciler/src/ReactFiberWorkLoop.js
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority(
    ImmediatePriority,
    commitRootImpl.bind(null, root, renderPriorityLevel),
  );
  // If there are passive effects, schedule a callback to flush them. This goes
  // outside commitRootImpl so that it inherits the priority of the render.
  if (rootWithPendingPassiveEffects !== null) {
    scheduleCallback(NormalPriority, () => {
      flushPassiveEffects();
      return null;
    });
  }
  return null;
}
```

在`commitRootImpl`有几次遍历`nextEffect`链表的过程

第一次是和一个生命周期有关： `getSnapshotBeforeUpdate`

> getSnapshotBeforeUpdate() 在最近一次渲染输出（提交到 DOM 节点）之前调用。它使得组件能在发生更改之前从 DOM 中捕获一些信息（例如，滚动位置）。此生命周期的任何返回值将作为参数传递给 componentDidUpdate()。

其实是把`stateNode`属性有关，由于这个生命周期用的很少， 我只放一点相关的代码大概的结构

ps: 还和`MemoComponent`有关，由于hooks现在先不讲，所以先略过

```js
// react-reconciler/src/ReactFiberCommitWork.js
function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber,
): void {
  switch (finishedWork.tag) {
    case SimpleMemoComponent: {
      commitHookEffectList(UnmountSnapshot, NoHookEffect, finishedWork);
      return;
    }
    case ClassComponent: {
      if (finishedWork.effectTag & Snapshot) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState,
          );
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      return;
    }
  }
}

```

```js
function commitRootImpl(root, renderPriorityLevel) {

  // 这里是已经完成了的current.alternate
  const finishedWork = root.finishedWork;
  const expirationTime = root.finishedExpirationTime;
  
  root.finishedWork = null;
  root.finishedExpirationTime = NoWork;

  invariant(
    finishedWork !== root.current,
    'Cannot commit the same tree as before. This error is likely caused by ' +
      'a bug in React. Please file an issue.',
  );

  // commitRoot never returns a continuation; it always finishes synchronously.
  // So we can clear these now to allow a new callback to be scheduled.
  root.callbackNode = null;
  root.callbackExpirationTime = NoWork;

  // 开始commit阶段  timer还是设计Perf之类的东西
  startCommitTimer();

  // Update the first and last pending times on this root. The new first
  // pending time is whatever is left on the root fiber.
  // 上次留下来的pendingTime 会作为这次的第一个pendingTime
  // 把子节点中优先级最高的和current中的expirationTime的优先级作比较
  const updateExpirationTimeBeforeCommit = finishedWork.expirationTime;
  const childExpirationTimeBeforeCommit = finishedWork.childExpirationTime;
  const firstPendingTimeBeforeCommit =
    childExpirationTimeBeforeCommit > updateExpirationTimeBeforeCommit
      ? childExpirationTimeBeforeCommit
      : updateExpirationTimeBeforeCommit;
  root.firstPendingTime = firstPendingTimeBeforeCommit;
  if (firstPendingTimeBeforeCommit < root.lastPendingTime) {
    // This usually means we've finished all the work, but it can also happen
    // when something gets downprioritized during render, like a hidden tree.
    // 完成了上一次的所有work，所以这次会把time改成下一次的第一个time
    // 也有可能是因为某种原因，有一些低优先级的render任务，比下一次需要更新的任务权重还要低一点的，
    // 比如说在隐藏域中的树的更新
    root.lastPendingTime = firstPendingTimeBeforeCommit;
  }

  // 在renderRoot的work步骤后就已经初始化为null了
  if (root === workInProgressRoot) {
    // We can reset these now that they are finished.
    workInProgressRoot = null;
    workInProgress = null;
    renderExpirationTime = NoWork;
  } else {
    // This indicates that the last root we worked on is not the same one that
    // we're committing now. This most commonly happens when a suspended root
    // times out.
  }

  // Get the list of effects.
  let firstEffect;
  // 这里没有effectTag，所以 = 0 而performedWork = 1
  // 所有比performedWork大的值都是带有副作用操作的，比如placement, update, callback等等
  if (finishedWork.effectTag > PerformedWork) {
    // A fiber's effect list consists only of its children, not itself. So if
    // the root has an effect, we need to add it to the end of the list. The
    // resulting list is the set that would belong to the root's parent, if it
    // had one; that is, all the effects in the tree including the root.
    // fiber的effect list是用来保存他的子节点的effect
    // 所以如果root节点有effect，我们必须添加到这个effect链表的最后
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // There is no effect on the root.
    // root没有effect
    firstEffect = finishedWork.firstEffect;
  }

  if (firstEffect !== null) {
    // 接上文 excutionContext被修改成了
    const prevExecutionContext = executionContext;
    // 在renderRoot里面它曾经被 | RenderContext
    // 这里与过来  表示进入commit阶段了
    executionContext |= CommitContext;
    let prevInteractions: Set<Interaction> | null = null;
    if (enableSchedulerTracing) {
      prevInteractions = __interactionsRef.current;
      __interactionsRef.current = root.memoizedInteractions;
    }

    // Reset this to null before calling lifecycles
    ReactCurrentOwner.current = null;

    // The commit phase is broken into several sub-phases. We do a separate pass
    // of the effect list for each phase: all mutation effects come before all
    // layout effects, and so on.

    // The first phase a "before mutation" phase. We use this phase to read the
    // state of the host tree right before we mutate it. This is where
    // getSnapshotBeforeUpdate is called.
    // 和debug有关
    startCommitSnapshotEffectsTimer();
    // 和不同环境的准备有关
    // 这个在reactDOM 中可以拿到一些当前节点的信息
    prepareForCommit(root.containerInfo);
    nextEffect = firstEffect;
    // 第一个循环
    do {
      if (__DEV__) {
        // 帮助错误调试的代码等等..
      } else {
        try {
          // commitBeforeMutationEffects 其实是一个nextEffect的循环
          // 每次会用nextEffect.nextEffect来遍历整个从firstEffect到lastEffect的链表
          // 上面也说了effect包含的是所有子节点的effect
          // 这里从reactDOM.render进来 自然需要包含所有子节点的挂载
          // 这个主要是和 `getSnapshotBeforeUpdate` 生命周期有关，
          // 并且根据nextEffect(其实就是个fiber)的tag判断，只有classComponent 才会处理
          commitBeforeMutationEffects();
        } catch (error) {
          invariant(nextEffect !== null, 'Should be working on an effect.');
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);
    // 和debug 、profiler有关略
    stopCommitSnapshotEffectsTimer();
    // profiler的不看
    if (enableProfilerTimer) {
      // Mark the current commit time to be shared by all Profilers in this
      // batch. This enables them to be grouped later.
      recordCommitTime();
    }

    // The next phase is the mutation phase, where we mutate the host tree.
    // 开始改变整个tree
    startCommitHostEffectsTimer();
    nextEffect = firstEffect;
    // 第二次循环
    // 执行完这个循环，整个domtree已经挂载到html上了，页面上已经有渲染了dom元素了
    do {
      if (__DEV__) {
       
      } else {
        try {
          // 开始mutate 同样是用nextEffect
          // 
          commitMutationEffects(renderPriorityLevel);
        } catch (error) {
          invariant(nextEffect !== null, 'Should be working on an effect.');
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);
    stopCommitHostEffectsTimer();
    // 这里有一个对selection选项恢复的函数
    // 因为placement的时候节点会摧毁再重新加入，所以有选项的时候要把选中的恢复
    resetAfterCommit(root.containerInfo);
    // 到此 已经更新完dom节点


    // The work-in-progress tree is now the current tree. This must come after
    // the mutation phase, so that the previous tree is still current during
    // componentWillUnmount, but before the layout phase, so that the finished
    // work is current during componentDidMount/Update.
    // 在mutation阶段完成以后再把finishedWork赋给root.current 
    root.current = finishedWork;

    // The next phase is the layout phase, where we call effects that read
    // the host tree after it's been mutated. The idiomatic use case for this is
    // layout, but class component lifecycles also fire here for legacy reasons.
    startCommitLifeCyclesTimer();
    nextEffect = firstEffect;
    do {
      if (__DEV__) {
        
      } else {
        try {
          // 主要是关于 commit完成后 生命周期及钩子的调用
          // 以及对updateQueue的清理，包括之前提到的`capturedQueue`的处理及update上的callback的调用
          commitLayoutEffects(root, expirationTime);
        } catch (error) {
          invariant(nextEffect !== null, 'Should be working on an effect.');
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);
    stopCommitLifeCyclesTimer();
    // 处理完以后必须要置空
    nextEffect = null;

    // Tell Scheduler to yield at the end of the frame, so the browser has an
    // opportunity to paint.
    requestPaint();

    if (enableSchedulerTracing) {
      __interactionsRef.current = ((prevInteractions: any): Set<Interaction>);
    }
    // 执行上下文也要变化
    executionContext = prevExecutionContext;
  } else {
    // No effects.
    // 这里是没有effect的情况
    root.current = finishedWork;
    // Measure these anyway so the flamegraph explicitly shows that there were
    // no effects.
    // TODO: Maybe there's a better way to report this.
    // 和timer相关的东西，也许是需要上报在没有effect时的效率记录，不太明白
    startCommitSnapshotEffectsTimer();
    stopCommitSnapshotEffectsTimer();
    if (enableProfilerTimer) {
      recordCommitTime();
    }
    startCommitHostEffectsTimer();
    stopCommitHostEffectsTimer();
    startCommitLifeCyclesTimer();
    stopCommitLifeCyclesTimer();
  }

  stopCommitTimer();


  // 被动的effect  不太懂 ，
  const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

  if (rootDoesHavePassiveEffects) {
    // This commit has passive effects. Stash a reference to them. But don't
    // schedule a callback until after flushing layout work.
    rootDoesHavePassiveEffects = false;
    rootWithPendingPassiveEffects = root;
    pendingPassiveEffectsExpirationTime = expirationTime;
    pendingPassiveEffectsRenderPriority = renderPriorityLevel;
  } else {
    // We are done with the effect chain at this point so let's clear the
    // nextEffect pointers to assist with GC. If we have passive effects, we'll
    // clear this in flushPassiveEffects.
    nextEffect = firstEffect;
    while (nextEffect !== null) {
      const nextNextEffect = nextEffect.nextEffect;
      nextEffect.nextEffect = null;
      nextEffect = nextNextEffect;
    }
  }

  // Check if there's remaining work on this root
  // 如果仍然有剩余的work需要做。这里好像只是一个记录？获取time以及优先级
  const remainingExpirationTime = root.firstPendingTime;
  if (remainingExpirationTime !== NoWork) {
    const currentTime = requestCurrentTime();
    const priorityLevel = inferPriorityFromExpirationTime(
      currentTime,
      remainingExpirationTime,
    );

    if (enableSchedulerTracing) {
      if (spawnedWorkDuringRender !== null) {
        const expirationTimes = spawnedWorkDuringRender;
        spawnedWorkDuringRender = null;
        for (let i = 0; i < expirationTimes.length; i++) {
          scheduleInteractions(
            root,
            expirationTimes[i],
            root.memoizedInteractions,
          );
        }
      }
    }

    scheduleCallbackForRoot(root, priorityLevel, remainingExpirationTime);
  } else {
    // If there's no remaining work, we can clear the set of already failed
    // error boundaries.
    // 否则做完并且没错误，把全局错误边界设置为null
    legacyErrorBoundariesThatAlreadyFailed = null;
  }

  if (enableSchedulerTracing) {
    if (!rootDidHavePassiveEffects) {
      // If there are no passive effects, then we can complete the pending interactions.
      // Otherwise, we'll wait until after the passive effects are flushed.
      // Wait to do this until after remaining work has been scheduled,
      // so that we don't prematurely signal complete for interactions when there's e.g. hidden work.
      finishPendingInteractions(root, expirationTime);
    }
  }
  // 和开发者工具有关的函数  在 ReactFiberDevToolsHook有关
  onCommitRoot(finishedWork.stateNode, expirationTime);

  // 这里计算未完成的root的最高优先级的重渲染，
  // 函数内的用来储存剩余work的优先级的变量remainingExpirationTime
  if (remainingExpirationTime === Sync) {
    // Count the number of times the root synchronously re-renders without
    // finishing. If there are too many, it indicates an infinite update loop.
    if (root === rootWithNestedUpdates) {
      nestedUpdateCount++;
    } else {
      nestedUpdateCount = 0;
      rootWithNestedUpdates = root;
    }
  } else {
    nestedUpdateCount = 0;
  }
  // 未捕获的错误会在这个阶段抛出，即不影响整个流程
  if (hasUncaughtError) {
    hasUncaughtError = false;
    const error = firstUncaughtError;
    firstUncaughtError = null;
    throw error;
  }

  if ((executionContext & LegacyUnbatchedContext) !== NoContext) {
    // This is a legacy edge case. We just committed the initial mount of
    // a ReactDOM.render-ed root inside of batchedUpdates. The commit fired
    // synchronously, but layout updates should be deferred until the end
    // of the batch.
    return null;
  }

  // If layout work was scheduled, flush it now.
  // 在调度阶段会把执行优先级为sync的callback放到callbackQueue[]这样一个数组里面
  // 在此时就会立即执行
  // 全局搜了一下，大部分能进入这个callbackQueue里的都是和root有关系
  // 或者是挂载在root.callbackNode上
  flushSyncCallbackQueue();
  return null;
}
```


------------------

`commitMutationEffects`

```js
function commitMutationEffects(renderPriorityLevel) {
  // TODO: Should probably move the bulk of this function to commitWork.
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;
    // 把effectTag |= ContentReset的情况是在 updateHostComponent 里 也就是需要把里面的text节点全部重置
    // effectTag 此时是Placement = 3 
    // 也就是下面两个都不会走进去
    // 具体的用法 ，等到遇到相应的再进去
    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }
    // 把effectTag |= Ref 的情况是在 completeWork的时候 或者updateHostComponent等等
    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // The following switch statement is only concerned about placement,
    // updates, and deletions. To avoid needing to add a case for every possible
    // bitmap value, we remove the secondary effects from the effect tag and
    // switch on that value.
    // 这里通过位运算 拿到应该执行的操作
    // 以我们进来的流程进入Placement为例
    let primaryEffectTag = effectTag & (Placement | Update | Deletion);
    switch (primaryEffectTag) {
      case Placement: {
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        // TODO: findDOMNode doesn't rely on this any more but isMounted does
        // and isMounted is deprecated anyway so we should be able to kill this.
        nextEffect.effectTag &= ~Placement;
        break;
      }
      case PlacementAndUpdate: {
        // Placement
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        nextEffect.effectTag &= ~Placement;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        commitDeletion(nextEffect, renderPriorityLevel);
        break;
      }
    }

    // TODO: Only record a mutation effect if primaryEffectTag is non-zero.
    recordEffect();

    resetCurrentDebugFiberInDEV();
    nextEffect = nextEffect.nextEffect;
  }
}
```

这里以一个commitPlacement为例继续追上去。最后在一系列的htmlElement操作的以后，以返回null跳出循环返回到上一步`commitMutationEffects`

```js
function commitPlacement(finishedWork: Fiber): void {
  if (!supportsMutation) {
    return;
  }

  // Recursively insert all host nodes into the parent.
  // 会循环fiber.return找到rootFiber
  const parentFiber = getHostParentFiber(finishedWork);

  // Note: these two variables *must* always be updated together.
  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  // 通过判断fiber的节点类型， 获取真正的节点信息
  // 取得它的父节点
  // 设置是否是容器的变量
  switch (parentFiber.tag) {
    case HostComponent:
    case HostRoot:
      // 在很早的createFiberRoot时就已经把containerInfo加上去了
      // 此时是一个root的节点信息
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
    case FundamentalComponent:
    // eslint-disable-next-line-no-fallthrough
    default:
  }
  // 同上 略过
  if (parentFiber.effectTag & ContentReset) {
    // Reset the text content of the parent before doing any insertions
    resetTextContent(parent);
    // Clear ContentReset from the effect tag
    parentFiber.effectTag &= ~ContentReset;
  }

  // 当finishedWork是root下的第一个子节点的时候，这是before = null
  const before = getHostSibling(finishedWork);
  // We only have the top Fiber that was inserted but we need to recurse down its
  // children to find all the terminal nodes.
  let node: Fiber = finishedWork;
  while (true) {
    const isHost = node.tag === HostComponent || node.tag === HostText;
    if (isHost || node.tag === FundamentalComponent) {
      const stateNode = isHost ? node.stateNode : node.stateNode.instance;
      if (before) {
        if (isContainer) {
          insertInContainerBefore(parent, stateNode, before);
        } else {
          insertBefore(parent, stateNode, before);
        }
      } else {
        // 从这里进入 就是htmlElement的基本操作了  执行完这一步 整个dom tree就会添加到节点中
        if (isContainer) {
          appendChildToContainer(parent, stateNode);
        } else {
          appendChild(parent, stateNode);
        }
      }
    } else if (node.tag === HostPortal) {
      // If the insertion itself is a portal, then we don't want to traverse
      // down its children. Instead, we'll get insertions from each child in
      // the portal directly.
    } else if (node.child !== null) {
      node.child.return = node;
      node = node.child;
      continue;
    }
    if (node === finishedWork) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === finishedWork) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```


----------------


接下来是第三个循环`commitLayoutEffects`. 包括关于生命周期


```js
function commitLayoutEffects(
  root: FiberRoot,
  committedExpirationTime: ExpirationTime,
) {
  // TODO: Should probably move the bulk of this function to commitWork.
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;

    if (effectTag & (Update | Callback)) {
      // 通过变量++ 记录effect
      recordEffect();
      const current = nextEffect.alternate;
      // 其实是commitLifeCycles的改名
      commitLayoutEffectOnFiber(
        root,
        current,
        nextEffect,
        committedExpirationTime,
      );
    }

    if (effectTag & Ref) {
      recordEffect();
      // attachRef
      commitAttachRef(nextEffect);
    }

    if (effectTag & Passive) {
      rootDoesHavePassiveEffects = true;
    }

    resetCurrentDebugFiberInDEV();
    nextEffect = nextEffect.nextEffect;
  }
}
```


追上面追进来`commitLifeCycles`（`commitLayoutEffectOnFiber`）。

这里主要是关于组件生命周期的，以最熟悉的`classComponent`来说主要是调用`componentDidMount`或者`componentDidUpdate`。以及会把`updateQueue`中的`capturedUpdate`里遍历执行`callback`，然后清空

而`FunctionComponent`,`ForwardRef`,`SimpleMemoComponent` 主要是对hooks链表的一些处理。比如`useEffect`



```js
// react-reconciler/src/ReactFiberCommitWork.js
function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedExpirationTime: ExpirationTime,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      // 对hooks链表的一些操作
      commitHookEffectList(UnmountLayout, MountLayout, finishedWork);
      break;
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.effectTag & Update) {
        // current === null 表示 didMount 否则就走didUpdate
        // todo  这里需要观察追踪一下何时把finishedWork
        if (current === null) {
          startPhaseTimer(finishedWork, 'componentDidMount');
          // 有一些prevState和memoziedState； prevProps和memoziedProps的比较
          // 不同完全相等， 这里todo去理解 什么时候把finishedWork.alternate置空的 在render 的时候

          // commit阶段最后就执行didMount
          instance.componentDidMount();
          stopPhaseTimer();
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          const prevState = current.memoizedState;
          startPhaseTimer(finishedWork, 'componentDidUpdate');
          
          instance.componentDidUpdate(
            prevProps,
            prevState,
            // __reactInternalSnapshotBeforeUpdate 在commitBeforeMutationLifeCycles里存储
            // 所以可以存进来
            instance.__reactInternalSnapshotBeforeUpdate,
          );
          stopPhaseTimer();
        }
      }
      const updateQueue = finishedWork.updateQueue;
      if (updateQueue !== null) {
        // We could update instance props and state here,
        // but instead we rely on them being set during last render.
        // TODO: revisit this when we implement resuming.
        // 这里有关于updateQueue里的capturedUpdate链表的操作。
        // 会遍历里面的所有capturedUpdate, 包括执行capturedUpdate上的callback，并且清空这个链表
        commitUpdateQueue(
          finishedWork,
          updateQueue,
          instance,
          committedExpirationTime,
        );
      }
      return;
    }
    case HostRoot: {
      const updateQueue = finishedWork.updateQueue;
      if (updateQueue !== null) {
        let instance = null;
        if (finishedWork.child !== null) {
          switch (finishedWork.child.tag) {
            case HostComponent:
              instance = getPublicInstance(finishedWork.child.stateNode);
              break;
            case ClassComponent:
              instance = finishedWork.child.stateNode;
              break;
          }
        }
        commitUpdateQueue(
          finishedWork,
          updateQueue,
          instance,
          committedExpirationTime,
        );
      }
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;

      // Renderers may schedule work to be done after host components are mounted
      // (eg DOM renderer may schedule auto-focus for inputs and form controls).
      // These effects should only be committed when components are first mounted,
      // aka when there is no current/alternate.
      if (current === null && finishedWork.effectTag & Update) {
        const type = finishedWork.type;
        const props = finishedWork.memoizedProps;
        commitMount(instance, type, props, finishedWork);
      }

      return;
    }
    case HostText: {
      // We have no life-cycles associated with text.
      return;
    }
    case HostPortal: {
      // We have no life-cycles associated with portals.
      return;
    }
    case Profiler: {
      if (enableProfilerTimer) {
        const onRender = finishedWork.memoizedProps.onRender;

        if (typeof onRender === 'function') {
          if (enableSchedulerTracing) {
            onRender(
              finishedWork.memoizedProps.id,
              current === null ? 'mount' : 'update',
              finishedWork.actualDuration,
              finishedWork.treeBaseDuration,
              finishedWork.actualStartTime,
              getCommitTime(),
              finishedRoot.memoizedInteractions,
            );
          } else {
            onRender(
              finishedWork.memoizedProps.id,
              current === null ? 'mount' : 'update',
              finishedWork.actualDuration,
              finishedWork.treeBaseDuration,
              finishedWork.actualStartTime,
              getCommitTime(),
            );
          }
        }
      }
      return;
    }
    case SuspenseComponent:
    case SuspenseListComponent:
    case IncompleteClassComponent:
    case FundamentalComponent:
      return;
    default: {
      invariant(
        false,
        'This unit of work tag should not have side-effects. This error is ' +
          'likely caused by a bug in React. Please file an issue.',
      );
    }
  }
}

```