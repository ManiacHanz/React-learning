## 关于Portal

> ReactDOM.createPortal(child, container)

`createPortal`只是一个`$$typeof`不同的fiber，没啥说的

```js
// react-dom/src/client/react-dom.js
function createPortal(
  children: ReactNodeList,
  container: DOMContainer,
  key: ?string = null,
) {
  // TODO: pass ReactDOM portal implementation as third argument
  return createPortalImpl(children, container, null, key);
}

// shared/ReactProtal.js
export function createPortal(
  children: ReactNodeList,
  containerInfo: any,
  // TODO: figure out the API for cross-renderer implementation.
  // 目前此参数为null
  implementation: any,
  key: ?string = null,
): ReactPortal {
  return {
    // This tag allow us to uniquely identify this as a React Portal
    $$typeof: REACT_PORTAL_TYPE,
    key: key == null ? null : '' + key,
    children,
    containerInfo,
    implementation,
  };
}
```

而判断`$$typeof`，主要是在reconciler`阶段, 也就是第十八节说的diff阶段

这次我们把不想关代码删了，只看`portal`的相关流程
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
  // ..

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


  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}

function reconcileSinglePortal(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  portal: ReactPortal,
  expirationTime: ExpirationTime,
): Fiber {
  const key = portal.key;
  let child = currentFirstChild;
  while (child !== null) {
    // TODO: If key === null and child.key === null, then this only applies to
    // the first item in the list.
    // 可复用和不可服用的不同处理方式
    if (child.key === key) {
      if (
        child.tag === HostPortal &&
        child.stateNode.containerInfo === portal.containerInfo &&
        child.stateNode.implementation === portal.implementation
      ) {
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(
          child,
          portal.children || [],
          expirationTime,
        );
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

  // 此方法会把fiber.tag = HostPortal
  const created = createFiberFromPortal(
    portal,
    returnFiber.mode,
    expirationTime,
  );
  created.return = returnFiber;
  return created;
}


export function createFiberFromPortal(
  portal: ReactPortal,
  mode: TypeOfMode,
  expirationTime: ExpirationTime,
): Fiber {
  const pendingProps = portal.children !== null ? portal.children : [];
  // tag  props  key  mode
  const fiber = createFiber(HostPortal, pendingProps, portal.key, mode);
  fiber.expirationTime = expirationTime;
  fiber.stateNode = {
    containerInfo: portal.containerInfo,
    pendingChildren: null, // Used by persistent updates
    implementation: portal.implementation,
  };
  return fiber;
}

```

打上了`HostPortal`的tag的fiber在后续的commitWork和completeWork阶段会有不同于其他的处理