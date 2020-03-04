### reconcileChildren 

`reconcileChildren`调和子节点，是一个相当长的方法，大概代码就有1100多行，所以当读提出来说

总结

这一章的作用主要是根据传进来的`fiber`以及`workInProgress`，对于children生成一个新的fiber对象，然后通过`child`和`return`属性，形成树结果


首先是`reconcileChildren`其实是调用的`ChildReconciler`方法

```js
// react-reconciler/src/ReactChildFiber.js
export const reconcileChildFibers = ChildReconciler(true);
export const mountChildFibers = ChildReconciler(false);
```

而同文件下的`ChildReconciler`是如下的结构， 为了看的清楚一些把方法的参数以及内容都删了，只留下了方法名和返回值类型

可以看到`ChildReconciler`执行后返回的是其在内部定义的`reconcileChildFibers`方法。而传入的参数代表`shouldTrackSideEffects`。和副作用有关。根据前面的内容我们也可以先猜测，应该是和`fiber`中的`effectTag`有关系。而其他的方法大概包括几类：
* delete
* update
* place
* create
* reconcile
这些很明显了就是根据之前在`update.tag`里面存的类型来判断进行操作。同时也会根据`element`的`$$typeof`来判断是否是`Protal``Fragment`等等。那我们就带着这些前面的积累，来看究竟是如何调和子节点的细节


```js
function ChildReconciler(shouldTrackSideEffects) {
  function deleteChild(): void {}

  function deleteRemainingChildren(): null {}

  function mapRemainingChildren(): Map<string | number, Fiber> {}

  function useFiber(): Fiber {}

  function placeChild(): number {}

  function placeSingleChild(): Fiber {}

  function updateTextNode() {}

  function updateElement(): Fiber {}

  function updatePortal(): Fiber {}

  function updateFragment(): Fiber {}

  function createChild(): Fiber | null {}

  function updateSlot(): Fiber | null {}

  function updateFromMap(): Fiber | null {}

  function warnOnInvalidKey(): Set<string> | null {}

  function reconcileChildrenArray(): Fiber | null {}

  function reconcileChildrenIterator(): Fiber | null {}

  function reconcileSingleTextNode(): Fiber {}

  function reconcileSingleElement(): Fiber {}

  function reconcileSinglePortal(): Fiber {}

  function reconcileChildFibers(): Fiber | null {}

  return reconcileChildFibers;
}
```


`reconcileChildFibers` 方法看起来很长一段代码，其实步骤很清晰：
1、判断传进来的children的`$$typeof`是不是`Fragment`，如果是的话就要取child.props.children作为后续分析的child来处理
2、根据child的`$$typeof`进行分类处理。有`SingleElement`或者是`SinglePortal`,有`TextNode`，以及可以循环的`ArrayNode`,`IterateNode`
3、执行`deleteRemainingChildren`

每一个方法应该都比较类似，我这里只留下我们走的流程的部分的代码。也就是执行了`reconcileSingleElement`，然后把结果传给`placeSingleChild`

```js
// This API will tag the children with the side-effect of the reconciliation
// itself. They will be added to the side-effect list as we pass through the
// children and the parent.
/*
* returnFiber： rootFiber
* currentFirstChild:  workInProgress    这里可以很通常的理解到是上面那个rootFiber在更新中的对应的workInProgress
* nextChildren:   workInProgress.memorizedState    element 这里是updateHostRoot以后得到的memorizedState，是一个reactElement对象
* expirationTime:   是从schedule调度，renderRoot一直传过来的Sync
*/
function reconcileChildFibers(
  returnFiber: Fiber,           
  currentFirstChild: Fiber | null,
  newChild: any,
  expirationTime: ExpirationTime,
): Fiber | null {
  // This function is not recursive.
  // If the top level item is an array, we treat it as a set of children,
  // not as a fragment. Nested arrays on the other hand will be treated as
  // fragment nodes. Recursion happens at the normal flow.

  // Handle top level unkeyed fragments as if they were arrays.
  // This leads to an ambiguity between <>{[...]}</> and <>...</>.
  // We treat the ambiguous cases above the same.
  // 如果遇到Fragment 可以看到是直接调用props.children来处理的，所以不会渲染Fragment或者<></>
  const isUnkeyedTopLevelFragment =
    typeof newChild === 'object' &&
    newChild !== null &&
    newChild.type === REACT_FRAGMENT_TYPE &&
    newChild.key === null;
  if (isUnkeyedTopLevelFragment) {
    newChild = newChild.props.children;
  }

  // Handle object types
  const isObject = typeof newChild === 'object' && newChild !== null;

  if (isObject) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            expirationTime,
          ),
        );
      case REACT_PORTAL_TYPE:
        // ...
    }
  }

  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}

```

`reconcileSingleElement`的主要作用就是根据传进去的参数，创造一个新的fiber对象，然后把需要的`key`,`return`,`mode`等属性附上去

这里的逻辑是，一开始的create Fiber的时候，只创造了`rootFiber`，而现在在之前搞定了update等等过程以后，开始把`workInProgress`里的`memoizedState`找出来，并且开始找它的child，并通过return 指向他的父级，最终形成虚拟dom tree

`createFiberFromElement`是通过传进去的`props`生成新的fiberNode，但是`elementType`，`expirationTime`这种比较重要的属性会通过props挂载上去。再返回回来的

而`placeSingleChild`就很简单，只是把返回回来的新`fiber`对象再加上一个`PLACEMENT`的`effectTag`，让后续的处理知道这里是替换操作


```js
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  expirationTime: ExpirationTime,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    // TODO: If key === null and child.key === null, then this only applies to
    // the first item in the list.
    if (child.key === key) {
      if (
        child.tag === Fragment
          ? element.type === REACT_FRAGMENT_TYPE
          : child.elementType === element.type 
      ) {
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(
          child,
          element.type === REACT_FRAGMENT_TYPE
            ? element.props.children
            : element.props,
          expirationTime,
        );
        existing.ref = coerceRef(returnFiber, child, element);
        existing.return = returnFiber;
        return existing;
      } else {
        deleteRemainingChildren(returnFiber, child);
        break;
      }
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      expirationTime,
      element.key,
    );
    created.return = returnFiber;
    return created;
  } else {
    // App 会直接走到这里
    // 构造一个 通过props创造回来的new Fiber对象，
    const created = createFiberFromElement(
      element,
      returnFiber.mode,
      expirationTime,
    );
    // 然后通过ref和return 形成一个完成的dom tree
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}


function placeSingleChild(newFiber: Fiber): Fiber {
  // This is simpler for the single child case. We only need to do a
  // placement for inserting new children.
  if (shouldTrackSideEffects && newFiber.alternate === null) {
    newFiber.effectTag = Placement;
  }
  return newFiber;
}
```


以上步骤都执行完，我们能看到返回出去的fiber对象会被赋值给workInProgress.child，形成完整的树


```js
// react-reconciler/src/ReactFiberBeginWork.js
// 这里是从5、的uploadHostRoot过来的
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderExpirationTime: ExpirationTime,
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime,
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderExpirationTime,
    );
  }
}
```

到这里整个`beginWork`的流程就算完了，我们需要回到前面去看`beginWork`的循环  