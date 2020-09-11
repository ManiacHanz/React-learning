所谓diff，其实就是react对整个vdom-tree的一个比较。单节点的比较就不说了，比较精彩的是列表的一个比较

直接抛出函数 `ChildReconciler`

之前在第六章也跟到过这部分，不过没有仔细，导致并不知其所以然。这里我们仔细看看。所谓的`ChildReconciler`大致分为几个部分

`delete`,`place`,`update`代表对元素的增删改操作

`reconcile`就代表对各个不同方案的diff

`mapRemainingChildren`代表对剩余的子元素进行循环，具体的后面会讲到

`useFiber`是一个clone workInProgress的函数。主要是用来保持现有`current fiber`可用，然后又需要更新`pending props`的时候用

返回的`reconcileChildFibers`可以理解成一个总领，来分配具体该进入哪种调和方案

```js
// react-reconciler/src/ReactChildFiber.js
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

  function reconcileChildrenArray(): Fiber | null {}

  function reconcileChildrenIterator(): Fiber | null {}

  function reconcileSingleTextNode(): Fiber {}

  function reconcileSingleElement(): Fiber {}

  function reconcileSinglePortal(): Fiber {}

  function reconcileChildFibers(): Fiber | null {}

  return reconcileChildFibers;
}
```


`reconcileChildFibers`

可以看到这只是一个简单的分类函数，按照不同的`$$typeof`来分门别类的处理diff操作，最后调用`deleteRemainingChildren`结束。

```js
// This API will tag the children with the side-effect of the reconciliation
  // itself. They will be added to the side-effect list as we pass through the
  // children and the parent.
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

    // 这里其实是把没有key属性的fragment<></>里面的提出来处理，忽略掉fragment本身
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

    // 根据$$typeof判断类型
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
          return placeSingleChild(
            reconcileSinglePortal(
              returnFiber,
              currentFirstChild,
              newChild,
              expirationTime,
            ),
          );
      }
    }

    if (typeof newChild === 'string' || typeof newChild === 'number') {
      return placeSingleChild(
        reconcileSingleTextNode(
          returnFiber,
          currentFirstChild,
          '' + newChild,
          expirationTime,
        ),
      );
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        expirationTime,
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        expirationTime,
      );
    }

    if (isObject) {
      throwOnInvalidObjectType(returnFiber, newChild);
    }

    if (typeof newChild === 'undefined' && !isUnkeyedTopLevelFragment) {
      // 错误处理  略过
    }

    // Remaining cases are all treated as empty.
    // 兜底的删除
    return deleteRemainingChildren(returnFiber, currentFirstChild);
  }
```

`deleteRemainingChildren`，很简单 ，遍历同一层的兄弟节点。调用`deleteChild`
这里的删除处理，其实是对fiber中的effect链表做处理。首先是对`returnFiber`--父级的链表后附加上要处理的fiber。同时把当前fiber的`effectTag`标记为`deletion`，同时把`nextEffect`设置为null. 毕竟到了commit阶段要被移除的节点后续不应该有更多的操作


```js
function deleteRemainingChildren(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
): null {
  if (!shouldTrackSideEffects) {
    // Noop.
    return null;
  }

  // TODO: For the shouldClone case, this could be micro-optimized a bit by
  // assuming that after the first child we've already added everything.
  let childToDelete = currentFirstChild;
  while (childToDelete !== null) {
    deleteChild(returnFiber, childToDelete);
    childToDelete = childToDelete.sibling;
  }
  return null;
}

function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  if (!shouldTrackSideEffects) {
    // Noop.
    return;
  }
  // Deletions are added in reversed order so we add it to the front.
  // At this point, the return fiber's effect list is empty except for
  // deletions, so we can just append the deletion to the list. The remaining
  // effects aren't added until the complete phase. Once we implement
  // resuming, this may not be true.
  const last = returnFiber.lastEffect;
  if (last !== null) {
    last.nextEffect = childToDelete;
    returnFiber.lastEffect = childToDelete;
  } else {
    returnFiber.firstEffect = returnFiber.lastEffect = childToDelete;
  }
  childToDelete.nextEffect = null;
  childToDelete.effectTag = Deletion;
}
```

下面先理一理简单的单个节点

`reconcileSingleTextNode`。对于`string`或者`number`等纯文本节点，react也并没有单纯的全部删除再替换，而是判断了是否至少有一个节点可以复用 -- 同类型，且只需要改变其中的内容。注意这里过来的`textContent`是string类型，因为外面传过来的时候用 `''+newChild` 传进来的。
这里的区别就在于`deleteRemainingChildren`函数中的参数时`currentFirstChild`还是`currentFirstChild.sibling`

```js
function reconcileSingleTextNode(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  textContent: string,
  expirationTime: ExpirationTime,
): Fiber {
  // There's no need to check for keys on text nodes since we don't have a
  // way to define them.
  if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
    // We already have an existing node so let's just update it and delete
    // the rest.
    deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
    const existing = useFiber(currentFirstChild, textContent, expirationTime);
    existing.return = returnFiber;
    return existing;
  }
  // The existing first child is not a text node so we need to create one
  // and delete the existing ones.
  deleteRemainingChildren(returnFiber, currentFirstChild);
  const created = createFiberFromText(
    textContent,
    returnFiber.mode,
    expirationTime,
  );
  created.return = returnFiber;
  return created;
}
```


`reconcileSingleElement` 对于单元素的diff判定其实和上面的文本的判定很类似，首先，该函数分为两个部分

1. 判断有没有可复用的元素
2. 创造一个新的element fiber返回回去

先说第一点，react做了哪些判断：
还是我们已经很熟悉的while遍历，一次把所有的元素遍历过来

```js
while(child !== null) {
  //...
  child = child.sibling
}
```

react能留下来复用的情况只有一种，`key`相同（包括null === null), 同时`element.type`也要相同。
但是有一点不同的是，react不一定需要第一个child和newChild相同。当第一个不同的时候，react只会调用`deleteChild`，而不是`deleteRemainingChildren`。这个场景就是比如说有`A,B,C`三个节点，也许中间会复用到`B`节点去生成`newB`节点，而不是直接删除所有节点

调用`deleteRemainingChildren`的情况也很简单，只要找到`key`相同的情况，不管能否复用(element type相同)，都会删除剩余的所有的节点

上面是第一步

第二步是当上面发现没有能够复用的节点以后，会创造一个新的`fiber node`并返回回去

`reconcileSinglePortal`和`SingleElement`基本相同，就不复述

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
          : child.elementType === element.type ||
            // Keep this check inline so it only runs on the false path:
            (__DEV__
              ? isCompatibleFamilyForHotReloading(child, element)
              : false)
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
    const created = createFiberFromElement(
      element,
      returnFiber.mode,
      expirationTime,
    );
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

`reconcileChildrenArray`的逻辑相对复杂很多，我先把几个重要节点列出来

能看到里面总共有2次`for`循环，两个条件判断并return， 一次map函数（返回一个Map数据结构）

也就是说，大体逻辑是，在**第一次循环**之后，有**两个条件**可以返回。如果都不满足，则会经历**一个map函数**，然后进入**第二次循环**，最后返回

`mapRemainingChildren`函数的作用先记住一下，是遍历`oldFiber`，队列里的剩余fibers，然后按照`key: fiber`的结构返回一个Map。这里如果`key`没有显示指定，就会用`index`来替代。所以如果能走到这一步的条件之一，至少是oldFiber !== null

然后第二轮的遍历就是对`mapRemainingChildren`返回的结果处理，也就是剩余的oldFiber

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  expirationTime: ExpirationTime,
): Fiber | null {
  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (newFiber === null) {
      break;
    }
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    return resultingFirstChild;
  }

  if (oldFiber === null) {
    return resultingFirstChild;
  }

  // Add all children to a key map for quick lookups.
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  // Keep scanning and use the map to restore deleted items as moves.
  for (; newIdx < newChildren.length; newIdx++) {
  }

  if (shouldTrackSideEffects) {
  }

  return resultingFirstChild;
}

```

接下来我们先看第一轮遍历, 第一次遍历完成或跳出的条件有3个
1. oldFiber === null    代表老的fiber列表遍历完成
2. newIdx = newChildren.length     代表新的fiber列表遍历完成
3. newFiber === null    代表某种情况下遇到不可复用的dom

而第三种情况我们需要先了解`updateSlot`函数
其中有几种情况会返回null
1. newFiber === null 或者 undefined 云云
2. 当newFiber 为文本节点 且 oldFiber.key !== null
3. 当newFiber 为单一节点， newFiber.key !== oldFiber.key   注：null === null
4. 当newFiber 为数组节点， 且oldFiber.key !== null 时
这4种情况下的跳出，极大可能会触发第二次遍历

placeChild在这里扮演一个很重要的角色，相当于处理数组遍历的指针，会把oldFiber.index, newIdx, lastPlacedIndex纷纷做出处理，然后给lastPlacedIndex赋新值。对几种情况我们分开解析
`const current = newFiber.alternate`
1. current === null:  这个时候说明newFiber是通过create创建的fiber对象，还没有workInProgress，所以一定是一个插入的操作。此时定义`newFiber.effectTag = Placement`，同时直接返回lastPlacedIndex
2. current !== null: 此时这个newFiber是通过update旧的fiber过来的，又分下面两种情况: 
  1. oldIndex < lastPlacedIndex: 此时表示是需要移动或替代的元素
  2. oldIndex >= lastPlacedIndex:  此时表示oldFiber可以保留在原地，并且把lastPlacedIndex赋值成oldIndex
举个例子 从abcd -> adcb ： ad会遵从2.2保留，而cb则会标记为移动

```js
let resultingFirstChild: Fiber | null = null;
let previousNewFiber: Fiber | null = null;

let oldFiber = currentFirstChild;
let lastPlacedIndex = 0;
let newIdx = 0;
let nextOldFiber = null;

for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  // 这里是对nextOldFiber 即下一个需要处理的fiber的一个赋值
  // 如果当前oldFiber的idx已经大于newIdx，则说明当前的oldFiber都需要被处理
  // 否则处理兄弟节点
  if (oldFiber.index > newIdx) {
    nextOldFiber = oldFiber;
    oldFiber = null;
  } else {
    nextOldFiber = oldFiber.sibling;
  }
  // updateSlot 在这里是个关键的函数
  // 以为它返回的newFiber如果等于null的话，则当前的循环会直接跳出，整个第一次循环就结束了
  const newFiber = updateSlot(
    returnFiber,
    oldFiber,
    newChildren[newIdx],
    expirationTime,
  );
  if (newFiber === null) {
    // TODO: This breaks on empty slots like null children. That's
    // unfortunate because it triggers the slow path all the time. We need
    // a better way to communicate whether this was a miss or null,
    // boolean, undefined, etc.
    // 需要把oldFiber赋值回来，因为很可能会经历第二次遍历，而第二次遍历需要继续走oldFiber.sibling的链表循环
    if (oldFiber === null) {
      oldFiber = nextOldFiber;
    }
    break;
  }
  if (shouldTrackSideEffects) {
    if (oldFiber && newFiber.alternate === null) {
      // We matched the slot, but we didn't reuse the existing fiber, so we
      // need to delete the existing child.
      // 无需复用oldFiber
      deleteChild(returnFiber, oldFiber);
    }
  }
  // placeChild是个很重要的函数，相当于处理列表遍历的指针
  lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
  if (previousNewFiber === null) {
    // TODO: Move out of the loop. This only happens for the first run.
    // resultingFirstChild变量是最终返回去的变量，也就是当前层级的第一个节点，剩下的都会以当前节点的sibling链表排布下去
    resultingFirstChild = newFiber;
  } else {
    // TODO: Defer siblings if we're not at the right index for this slot.
    // I.e. if we had null values before, then we want to defer this
    // for each null value. However, we also don't want to call updateSlot
    // with the previous one.
    // 通过这个链表，形成新的数组排列顺序
    previousNewFiber.sibling = newFiber;
  }
  previousNewFiber = newFiber;
  // 开始循环下一个要处理的fiber
  oldFiber = nextOldFiber;
}
```

由此我们第一次的遍历到此结束，来看看两个能跳出的条件。 遍历完成没有跳出的条件就是，旧的数据列表都可以复用

这两个其实很明显了

`newIdx === newChildren.length` 代表新数组的长度已经遍历完成，此时只需要只需要把多余的oldFiber后面的兄弟节点删除即可

`oldFiber === null` 代表旧数组的长度已经遍历完成，此时剩余新的列表的数据相当于都是插入的节点，只需要重新create出来然后插入即可

```js
if (newIdx === newChildren.length) {
  // We've reached the end of the new children. We can delete the rest.
  deleteRemainingChildren(returnFiber, oldFiber);
  return resultingFirstChild;
}

if (oldFiber === null) {
  // If we don't have any more existing children we can choose a fast path
  // since the rest will all be insertions.
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = createChild(
      returnFiber,
      newChildren[newIdx],
      expirationTime,
    );
    if (newFiber === null) {
      continue;
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      // TODO: Move out of the loop. This only happens for the first run.
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
  }
  return resultingFirstChild;
}
```

那么我们直接看第二次循环

能进入这里，也就是第一次循环没有完成，即newIndex < newChildren.length，且oldFiber.sibling !== null (还有下一个)。也就是说两个列表都还有剩余。
那么简单的想，我们应该把剩余的项再次进入第一个循环，重新再找可复用的节点。但这样的话，第一会存在严重的效率问题，第二，仍然可能再次被跳出，那循环的次数就会非常多。
所以react在这里使用了`mapRemainingChildren`函数，直接把剩余的oldFibers变成一个{key: fiber}的Map，后面直接通过key去寻找可复用的fiber

`mapRemainingChildren`的源码很简单

```js
function mapRemainingChildren(
  returnFiber: Fiber,
  currentFirstChild: Fiber,
): Map<string | number, Fiber> {
  // Add the remaining children to a temporary map so that we can find them by
  // keys quickly. Implicit (null) keys get added to this set with their index
  // instead.
  const existingChildren: Map<string | number, Fiber> = new Map();

  let existingChild = currentFirstChild;
  while (existingChild !== null) {
    if (existingChild.key !== null) {
      existingChildren.set(existingChild.key, existingChild);
    } else {
      existingChildren.set(existingChild.index, existingChild);
    }
    existingChild = existingChild.sibling;
  }
  return existingChildren;
}
```

接下来就是最后的第二次循环，革命马上就成功了

`updateFromMap` 通过Map结构拿到对应index下标的Fiber，然后根据key和$$typeof的类型去判断是复用还是造新fiber返回回来。相当于一个策略函数

因为新列表还没遍历完。所以以新列表为主，继续遍历。 后续的逻辑和上面基本一致，到这里 所有的diff就都看完了

```js
// Add all children to a key map for quick lookups.
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

// Keep scanning and use the map to restore deleted items as moves.
for (; newIdx < newChildren.length; newIdx++) {
  // 通过existingChildren Map映射去拿到可复用的元素
  const newFiber = updateFromMap(
    existingChildren,
    returnFiber,
    newIdx,
    newChildren[newIdx],
    expirationTime,
  );
  if (newFiber !== null) {
    if (shouldTrackSideEffects) {
      if (newFiber.alternate !== null) {
        // The new fiber is a work in progress, but if there exists a
        // current, that means that we reused the fiber. We need to delete
        // it from the child list so that we don't add it to the deletion
        // list.
        // 这里删除说明在updateFormMap的过程中复用了原来的元素，这里删掉，保证不会把这个复用的添加到删除列表
        // 也就是给他的effectTag增加delection
        // 因为在外层最后有个兜底的删除
        existingChildren.delete(
          newFiber.key === null ? newIdx : newFiber.key,
        );
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
  }
}
```