
前面都只看了类组件，那么函数组件，比如Hook又是怎么进入更新流程的呢

我们还是随着之前的流程找到`workLoop`在找到`beginWork`，里面有下面这段代码

```js
// react-reconciler/src/ReactFiberBeginWork.js
switch (workInProgress.tag) {
  case FunctionComponent: {
      // 这里的type就是函数本身了
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      );
    }
}
```

然后我们跟着流程进入`updateFunctionComponent`, 由于我们只关注hooks相关，所以我们只看相关代码

```js
function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderExpirationTime,
) {
 
  let context;
  let nextChildren;
  prepareToReadContext(workInProgress, renderExpirationTime);
  if (__DEV__) {
    ReactCurrentOwner.current = workInProgress;
    setCurrentPhase('render');
    nextChildren = renderWithHooks(
      current,
      workInProgress,
      Component,
      nextProps,
      context,
      renderExpirationTime,
    );
  }
}
```

把其他代码删除掉以后能发现这里有个`renderWithHooks`方法，会带我们找到`packages/react-reconciler/src/ReactFiberHooks.js`文件中。在这个文件里我们能找到下面hooks相关的逻辑，从这里开始进入正题



```js
export function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  props: any,
  refOrContext: any,
  nextRenderExpirationTime: ExpirationTime,
): any {
  renderExpirationTime = nextRenderExpirationTime;
  // 当前模块的全局对象，相当于那边的workInProgress
  currentlyRenderingFiber = workInProgress;
  // hook 其实也是以一个链表保存
  // 函数式组件里 memoizedState就是用来保存hooks的状态了
  nextCurrentHook = current !== null ? current.memoizedState : null;

  // 在进行update过程中nextCurrentHook会用来保存hook list的下一个对应的节点
  // 这里没有就是直接用mount对应的hook，有就用update的hook
  // 这里就可以理解成调用的hook就是这上面的hook了。具体的可以从React入口去看
  // 那里调用的时候其实就是从这个对象上去取的
  // 所以我们往下看，看这个HooksDispatcherOnMount是何物
  ReactCurrentDispatcher.current =
    nextCurrentHook === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
  // 由于Component在外面传进来的时候是传的fiber.type
  // 对于函数式组件来说type就是这个函数本身
  // 所以执行它就能得到返回值，也就是这个vdom
  let children = Component(props, refOrContext);

  // ... 省略若干

  return children;
}
```

我们可以看到其实就是一个包含所有hooks的对象，所以我们就直接挑一个简单的来看吧，比如常用的`useState`

```js
const HooksDispatcherOnMount: Dispatcher = {
  readContext,
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  useDebugValue: mountDebugValue,
  useResponder: createResponderListener,
};

const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,

  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  useDebugValue: updateDebugValue,
  useResponder: createResponderListener,
};

```


这里flow的类型写出来以后就感觉有点乱，要仔细看一下

首先useState()参数可以穿一个值，或者是一个函数，这个函数的返回值相当于是传的这个值，如果是函数，它会先执行一遍。挂载到通过`mountWorkInProgressHook`方法执行出来的对象上

`mountWorkInProgressHook`方法主要是创建了一个初始化对象，里面有`memoizedState` `queue` `next` `baseUpdate` 等等这些属性。看了那么久源码大概也能猜到这些是什么意思了吧。然后会判断当前文件中的变量`workInProgressHook`是否为空，后续的操作就是很典型的对链表的操作，然后把当前的这个`workInProgressHook`返回回去。也就是说这个变量最后会被组成一个链表

好，再跳出来看。`mountState`把传入的初始值挂载`hook.memoizedState`和`hook.baseState`上，然后把`hook.queue`初始化一个对象，里面有`reducer`和`state`. 这里就能看到redux开发者的一个执着。最后初始化好`hook.queue.dispatch`分发函数即完毕。返回的第二位其实就是这个dispatch

屡一下也就是说，最后我们改变视图渲染的时候其实就是触发了这个`dispatch`函数，那我们就思路很清晰了，我们只需要进去看下这个`dispatch`是什么就可以了

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    last: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchAction.bind(
    null,
    // Flow doesn't know this is non-null, but we do.
    ((currentlyRenderingFiber: any): Fiber),
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}



function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,

    baseState: null,
    queue: null,
    baseUpdate: null,

    next: null,
  };
  // 典型的链表操作
  if (workInProgressHook === null) {
    // This is the first hook in the list
    firstWorkInProgressHook = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

函数很长，如果不想看，我们直接看到最后，熟悉的老面孔来了. `scheduleWork`，又进入render阶段了，马上又会走`workLoop`，走`beginWork`了。所以我们猜测也能想到，在这之前，react就已经把该fiber需要作出的修改搞定了，那么我们带着这样的想法来看这个方法

```js
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  const alternate = fiber.alternate;
  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // This is a render phase update. Stash it in a lazily-created map of
    // queue -> linked list of updates. After this render pass, we'll restart
    // and apply the stashed updates on top of the work-in-progress hook.
    didScheduleRenderPhaseUpdate = true;
    const update: Update<S, A> = {
      expirationTime: renderExpirationTime,
      suspenseConfig: null,
      action,
      eagerReducer: null,
      eagerState: null,
      next: null,
    };
  
    if (renderPhaseUpdates === null) {
      renderPhaseUpdates = new Map();
    }
    const firstRenderPhaseUpdate = renderPhaseUpdates.get(queue);
    if (firstRenderPhaseUpdate === undefined) {
      renderPhaseUpdates.set(queue, update);
    } else {
      // Append the update to the end of the list.
      let lastRenderPhaseUpdate = firstRenderPhaseUpdate;
      while (lastRenderPhaseUpdate.next !== null) {
        lastRenderPhaseUpdate = lastRenderPhaseUpdate.next;
      }
      lastRenderPhaseUpdate.next = update;
    }
  } else {
    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }

    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update: Update<S, A> = {
      expirationTime,
      suspenseConfig,
      action,
      eagerReducer: null,
      eagerState: null,
      next: null,
    };

    // Append the update to the end of the list.
    const last = queue.last;
    if (last === null) {
      // This is the first update. Create a circular list.
      update.next = update;
    } else {
      const first = last.next;
      if (first !== null) {
        // Still circular.
        update.next = first;
      }
      last.next = update;
    }
    queue.last = update;

    if (
      fiber.expirationTime === NoWork &&
      (alternate === null || alternate.expirationTime === NoWork)
    ) {
      // The queue is currently empty, which means we can eagerly compute the
      // next state before entering the render phase. If the new state is the
      // same as the current state, we may be able to bail out entirely.
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        let prevDispatcher;
        if (__DEV__) {
          prevDispatcher = ReactCurrentDispatcher.current;
          ReactCurrentDispatcher.current = InvalidNestedHooksDispatcherOnUpdateInDEV;
        }
        try {
          const currentState: S = (queue.lastRenderedState: any);
          const eagerState = lastRenderedReducer(currentState, action);
          // Stash the eagerly computed state, and the reducer used to compute
          // it, on the update object. If the reducer hasn't changed by the
          // time we enter the render phase, then the eager state can be used
          // without calling the reducer again.
          update.eagerReducer = lastRenderedReducer;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            // Fast path. We can bail out without scheduling React to re-render.
            // It's still possible that we'll need to rebase this update later,
            // if the component re-renders for a different reason and by that
            // time the reducer has changed.
            return;
          }
        } catch (error) {
          // Suppress the error. It will throw again in the render phase.
        } finally {
          if (__DEV__) {
            ReactCurrentDispatcher.current = prevDispatcher;
          }
        }
      }
    }
    if (__DEV__) {
      // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
      if ('undefined' !== typeof jest) {
        warnIfNotScopedWithMatchingAct(fiber);
        warnIfNotCurrentlyActingUpdatesInDev(fiber);
      }
    }
    scheduleWork(fiber, expirationTime);
  }
}

```