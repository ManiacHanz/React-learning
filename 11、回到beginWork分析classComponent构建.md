
所以在work阶段以前，react所有的操作都停留在fiber对象的处理阶段，不论是对于expirationTime的计算或者对于update的操作更新，都不涉及对于不同的component的类型的处理。只有进入work阶段开始，遍历树的时候，才会根据`workInProgress.tag`的不同值，对应不同的`component`类型进行处理。

这里先从最常用的`classComponent`的处理开始看，后面会延伸到其他的类型的component

把`beginWork`的最简化，进入`current === null`的状态，一切就显得非常清晰

```js
// react-reconciler/src/ReactFiberBeginWork
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  const updateExpirationTime = workInProgress.expirationTime;

  workInProgress.expirationTime = NoWork;

  switch (workInProgress.tag) {
    case ClassComponent: {
      // 这里展现了workInProgress.type的作用
      // 从mount过程过来这里的type等等其实是存在fiberRoot的update链表里面
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      );
    }
  }
}
```


这里可以看到 `updateClassComponent`主要是对`ClassInstance`的操作，从`construct`到`mount`，或者是`resume`和`update`。
最后完成`finish`

接下来就一个一个分析，也许后面可以看出许多相似的地方，对于分析其他的component就有利

```js
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps,
  renderExpirationTime: ExpirationTime,
) {
  // Push context providers early to prevent context stack mismatches.
  // During mounting we don't know the child context yet as the instance doesn't exist.
  // We will invalidate the child context in finishClassComponent() right after rendering.
  // 这里和老的context使用方法有关。通过类声明的childContextTypes获取context里的内容
  let hasContext;
  if (isLegacyContextProvider(Component)) {
    hasContext = true;
    // 推入context栈 和老的context处理方法有关
    pushLegacyContextProvider(workInProgress);
  } else {
    hasContext = false;
  }
  prepareToReadContext(workInProgress, renderExpirationTime);
  // 获取stateNode
  const instance = workInProgress.stateNode;
  let shouldUpdate;
  if (instance === null) {
    if (current !== null) {
      // 根据注释
      // 没有instance却有current的情况，也就是有fiber却没有mount节点的情况
      // 只会发生在 non-concurrent树 中的suspended组件
      // 这里要断掉alternate和current之间的关联
      // 可以忽略这个条件判断
      // An class component without an instance only mounts if it suspended
      // inside a non- concurrent tree, in an inconsistent state. We want to
      // tree it like a new mount, even though an empty version of it already
      // committed. Disconnect the alternate pointers.
      current.alternate = null;
      workInProgress.alternate = null;
      // Since this is conceptually a new fiber, schedule a Placement effect
      workInProgress.effectTag |= Placement;
    }
    // In the initial pass we might need to construct the instance.
    constructClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
    mountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
    // 主要是用给finishClassComponent用的
    shouldUpdate = true;
  } else if (current === null) {
    // In a resume, we'll already have an instance we can reuse.
    shouldUpdate = resumeMountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
  } else {
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
  }
  const nextUnitOfWork = finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderExpirationTime,
  );
  
  return nextUnitOfWork;
}

```

先来看`constructClassInstance`，发现这个方法主要是实例化`Component`，这里包括实例`props`,`context`,`state`以及`updater`，另外就是把当前的`wIP`和`instance`连接起来，当然是通过`stateNode`


```js
function constructClassInstance(
  workInProgress: Fiber,
  ctor: any,
  props: any,
  renderExpirationTime: ExpirationTime,
): any {
  // 还是关于contextType以及childContextType的处理
  // 前者是provider 后者是consumer
  let isLegacyContextConsumer = false;
  let unmaskedContext = emptyContextObject;
  let context = emptyContextObject;
  const contextType = ctor.contextType;

  if (typeof contextType === 'object' && contextType !== null) {
    context = readContext((contextType: any));
  } else if (!disableLegacyContext) {
    unmaskedContext = getUnmaskedContext(workInProgress, ctor, true);
    const contextTypes = ctor.contextTypes;
    isLegacyContextConsumer =
      contextTypes !== null && contextTypes !== undefined;
    context = isLegacyContextConsumer
      ? getMaskedContext(workInProgress, unmaskedContext)
      : emptyContextObject;
  }
  
  // 新建实例。这里的ctor就是我们上一章从BaseComponent文件里面看到的声明的Component类
  // 这里关键的updater还没传进去 也就是用的默认的，所以接下来我们需要着重关注updater到底是什么，为什么可以setState
  const instance = new ctor(props, context);
  const state = (workInProgress.memoizedState =
    instance.state !== null && instance.state !== undefined
      ? instance.state
      : null);
  // 两个作用
  // 1, 把workInProgress的state指向指到instance，instance._reactInternalFiber指向workInProgress
  // 2, 用classComponentUpdater赋值到instance.updater，也就是上面需要关注的部分
  adoptClassInstance(workInProgress, instance);

  // Cache unmasked context so we can avoid recreating masked context unless necessary.
  // ReactFiberContext usually updates this cache but can't for newly-created instances.
  if (isLegacyContextConsumer) {
    cacheContext(workInProgress, unmaskedContext, context);
  }

  return instance;
}


function adoptClassInstance(workInProgress: Fiber, instance: any): void {
  // 这里增加了updater，也就是说setState和forceUpdate是调用的classComponentUpdater里的方法
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;
  // The instance needs access to the fiber so that it can schedule updates
  // statenode实例需要一个属性能调度到workInProgress fiber
  setInstance(instance, workInProgress);
}

// setInstance
export function set(key, value) {
  key._reactInternalFiber = value;
}
```

看看`classComponentUpdater`的实现，可以看出来`setState`等方法是从哪里介入的

其实调用`setState`以后，会先通过expirationTime的计算，来获取更新的权重，然后通过`createUpdate`的初始化一个update，最后进入正常的enqueueUpdate以及scheduleWork过程。这后面的流程使我们之前没有看过的 非`Sync`的更新

```js
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    // 三个参数分别是classComponent实例，参数和回调
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    // 计算expirationTime
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );
    // 同第二节，也就是说要初始化一个update
    const update = createUpdate(expirationTime, suspenseConfig);
    // setState的payload就是setState的payload
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      // callback也会放入update上面
      update.callback = callback;
    }
    // https://github.com/facebook/react/pull/15650
    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
  // 没有暴露出来的方法，只在`willMount`和`willRecieveProps`里面使用
  // 判断的使用场景都是oldState !== instance.state
  // 也就是说判断在这两个钩子以后改变了state引用地址，比如用this.state = 
  // 会报警告同时调用这个方法
  enqueueReplaceState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.tag = ReplaceState;
    update.payload = payload;

    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'replaceState');
      }
      update.callback = callback;
    }

    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
  // 除了update.tag = ForceUpdate以外其他的和普通的setState一模一样
  // 相当于setState({})
  // 而这个ForceUpdate的作用主要是增加一个全局变量的开关，在resumeClassComponent和updateClassComponent的函数中
  // 判断是否应该进行update（两个条件：1、tag = ForceUpdate, 或 2、新老props和state的比较）
  // 而后调用willUpdate的钩子
  // 总的来说就可以相当于多了一个方法触发classComponent更新
  enqueueForceUpdate(inst, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }

    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
};
```

--------------

看完了`constructor`在来看看如何`mount`



```js

// Invokes the mount life-cycles on a previously never rendered instance.
function mountClassInstance(
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderExpirationTime: ExpirationTime,
): void {
  // 通过construct方法后，这里已经可以直接在stateNode上拿到实例了
  const instance = workInProgress.stateNode;
  // 赋值
  instance.props = newProps;
  instance.state = workInProgress.memoizedState;
  instance.refs = emptyRefsObject;

  const contextType = ctor.contextType;
  if (typeof contextType === 'object' && contextType !== null) {
    instance.context = readContext(contextType);
  } else if (disableLegacyContext) {
    instance.context = emptyContextObject;
  } else {
    const unmaskedContext = getUnmaskedContext(workInProgress, ctor, true);
    instance.context = getMaskedContext(workInProgress, unmaskedContext);
  }

  // 拿到需要更新的updateQueue
  let updateQueue = workInProgress.updateQueue;
  if (updateQueue !== null) {
    processUpdateQueue(
      workInProgress,
      updateQueue,
      newProps,
      instance,
      renderExpirationTime,
    );
    instance.state = workInProgress.memoizedState;
  }
  // 静态方法，直接从类上拿
  const getDerivedStateFromProps = ctor.getDerivedStateFromProps;
  if (typeof getDerivedStateFromProps === 'function') {
    // 相当于把getDerivedStateFromProps返回的对象和prevState进行浅合并，然后挂到wIP.memoziedState上以及非空的updateQueue.baseState上
    applyDerivedStateFromProps(
      workInProgress,
      ctor,
      getDerivedStateFromProps,
      newProps,
    );
    instance.state = workInProgress.memoizedState;
  }

  // In order to support react-lifecycles-compat polyfilled components,
  // Unsafe lifecycles should not be invoked for components using the new APIs.
  // 注意：新老更新的api不能同时调用
  if (
    typeof ctor.getDerivedStateFromProps !== 'function' &&
    typeof instance.getSnapshotBeforeUpdate !== 'function' &&
    (typeof instance.UNSAFE_componentWillMount === 'function' ||
      typeof instance.componentWillMount === 'function')
  ) {
    callComponentWillMount(workInProgress, instance);
    // If we had additional state updates during this life-cycle, let's
    // process them now.
    updateQueue = workInProgress.updateQueue;
    // processUpdateQueue是一个相对重要的对于updateQueue处理的方法，
    // 如果忘了可以返回第5看看，大概就是遍历update链表去找到权重高的更新
    // 同时会处理一些如baseState，firstUpdate等属性
    if (updateQueue !== null) {
      processUpdateQueue(
        workInProgress,
        updateQueue,
        newProps,
        instance,
        renderExpirationTime,
      );
      instance.state = workInProgress.memoizedState;
    }
  }
  // 把effectTag 复合一个update属性
  // 只有复合了这个属性，才会在commit阶段中调用componentDidMount
  // 见ReactFiberCommitWork
  //  if (finishedWork.effectTag & Update) {
  //       if (current === null) {
  //         startPhaseTimer(finishedWork, 'componentDidMount');
  //         // We could update instance props and state here,
  //         // but instead we rely on them being set during last render.
  //         // TODO: revisit this when we implement resuming.
  //         instance.componentDidMount();
  //         stopPhaseTimer();
  //       } else {
  //         const prevProps =
  //           finishedWork.elementType === finishedWork.type
  //             ? current.memoizedProps
  //             : resolveDefaultProps(finishedWork.type, current.memoizedProps);
  //         const prevState = current.memoizedState;
  //         startPhaseTimer(finishedWork, 'componentDidUpdate');
  //         // We could update instance props and state here,
  //         // but instead we rely on them being set during last render.
  //         // TODO: revisit this when we implement resuming.
  //         instance.componentDidUpdate(
  //           prevProps,
  //           prevState,
  //           instance.__reactInternalSnapshotBeforeUpdate,
  //         );
  //         stopPhaseTimer();
  //       }
  //     }
  if (typeof instance.componentDidMount === 'function') {
    workInProgress.effectTag |= Update;
  }
}
```