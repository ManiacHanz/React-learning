
放一个大佬的事件总结，写的很明确[juejin](https://juejin.im/post/5d44e3745188255d5861d654)


首先放上调用堆栈的顺序图，让大家一眼能看出在哪个阶段进行事件注册的

![事件注册调用堆栈]('./image/13、react事件注册堆栈.jpg')

其实理一下前面学习的逻辑，react针对不同平台实现不同的dom渲染已经事件处理，肯定只有在commit或commit阶段以后去实现事件的绑定注入。而前面关于fiber的一切调度的处理都和平台无关，不可能有关于事件注入的东西

再想一想，这其实是前面欠的债..

把废代码去掉，我们主要看看 `HostComponent` 

```js
// react-reconciler/src/ReactFiberCompleteWork.js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
      break;
    case LazyComponent:
      break;
    case SimpleMemoComponent:
    case FunctionComponent:
      break;
    case ClassComponent: 
      break;
    case HostRoot: {
      popHostContainer(workInProgress);
      popTopLevelLegacyContextObject(workInProgress);
      const fiberRoot = (workInProgress.stateNode: FiberRoot);
      if (fiberRoot.pendingContext) {
        fiberRoot.context = fiberRoot.pendingContext;
        fiberRoot.pendingContext = null;
      }
      if (current === null || current.child === null) {
        // If we hydrated, pop so that we can delete any remaining children
        // that weren't hydrated.
        popHydrationState(workInProgress);
        // This resets the hacky state to fix isMounted before committing.
        // TODO: Delete this when we delete isMounted and findDOMNode.
        workInProgress.effectTag &= ~Placement;
      }
      updateHostContainer(workInProgress);
      break;
    }
    case HostComponent: {
      // 和context有关系
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      // 这里的workInProgress是全局的wIP
      // 要注意这里进来的时候这里属于什么。 通过beginWork深度遍历fiber树以后找到第一个没有child的fiber节点
      const type = workInProgress.type;
      // 不等于null表示已经构建完成一次了
      // 第一次reactDOM.mount进来的时候还是null
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );
        // FlareApi 是react事件系统中的扩展
        // 是一个实验性的功能，用来实现跨端及跨平台的事件处理
        // The idea is to extend React's event system to include high-level events that allow for consistent cross-device and cross-platform behavior.
        if (enableFlareAPI) {
          const prevListeners = current.memoizedProps.listeners;
          const nextListeners = newProps.listeners;
          const instance = workInProgress.stateNode;
          if (prevListeners !== nextListeners) {
            updateEventListeners(
              nextListeners,
              instance,
              rootContainerInstance,
              workInProgress,
            );
          }
        }

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {

        const currentHostContext = getHostContext();
        // TODO: Move createInstance to beginWork and keep it on a context
        // "stack" as the parent. Then append children as we go in beginWork
        // or completeWork depending on we want to add then top->down or
        // bottom->up. Top->down is faster in IE11.
        // 对于hydrate fiber的一个全局变量，相当于hydrate里的wIP
        let wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          // hydrate渲染 略
        } else {
          // 这里的createInstance就会变成分成多端的instance实例。包括后来的动态编译的语言应该是从这里开始进行的分流
          // 这里我们先不看react-native，先看react-dom端的
          // 这里生成的是相当于就是真实的dom了，并且能通过__reactInternalInstance和__reactEventHandlers访问fiber和pendingProps
          let instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );

          appendAllChildren(instance, workInProgress, false, false);

          // Certain renderers require commit-time effects for initial mount.
          // (eg DOM renderer supports auto-focus for certain elements).
          // Make sure such renderers get scheduled for later work.
          if (
            finalizeInitialChildren(
              instance,
              type,
              newProps,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress);
          }
          workInProgress.stateNode = instance;
        }

        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress);
        }
      }
      break;
    }
    case HostText: 
      break;
    case ForwardRef:
      break;
    case SuspenseComponent: 
      break;
    case Fragment:
      break;
    case Mode:
      break;
    case Profiler:
      break;
    case HostPortal:
      break;
    case ContextProvider:
      // Pop provider fiber
      break;
    case ContextConsumer:
      break;
    case MemoComponent:
      break;
    case IncompleteClassComponent: 
      break;
    case DehydratedSuspenseComponent: 
      break;
    case SuspenseListComponent: 
      break;
    case FundamentalComponent: 
      break;
    default:
  }

  return null;
}
```



```js
// react-dom/src/client/ReactDOMHostConfig.js
export function createInstance(
  type: string,
  props: Props,
  rootContainerInstance: Container,
  hostContext: HostContext,
  internalInstanceHandle: Object,
): Instance {
  let parentNamespace: string;
  if (__DEV__) {
    // TODO: take namespace into account when validating.
    const hostContextDev = ((hostContext: any): HostContextDev);
    validateDOMNesting(type, null, hostContextDev.ancestorInfo);
    if (
      typeof props.children === 'string' ||
      typeof props.children === 'number'
    ) {
      const string = '' + props.children;
      const ownAncestorInfo = updatedAncestorInfo(
        hostContextDev.ancestorInfo,
        type,
      );
      validateDOMNesting(null, string, ownAncestorInfo);
    }
    parentNamespace = hostContextDev.namespace;
  } else {
    parentNamespace = ((hostContext: any): HostContextProd);
  }
  const domElement: Instance = createElement(
    type,
    props,
    rootContainerInstance,
    parentNamespace,
  );
  // 这两步是把domElement以及pendingProps缓存到element的属性上
  // const internalInstanceKey = '__reactInternalInstance$' + randomKey;
  // const internalEventHandlersKey = '__reactEventHandlers$' + randomKey;
  precacheFiberNode(internalInstanceHandle, domElement);
  updateFiberProps(domElement, props);
  // 返回完成的domElement  这里就是vdom变成html dom的过程
  return domElement;
}

// react-dom/src/client/ReactDOMComponentTree.js
export function updateFiberProps(node, props) {
  node[internalEventHandlersKey] = props;
}

export function precacheFiberNode(hostInst, node) {
  node[internalInstanceKey] = hostInst;
}
```


```js
// react-reconciler/src/ReactFiberCompleteWork.js
// 通过遍历整颗fiber树，把所有的真实dom挂在上去
// 链接的过程是按照父子相连，然后在遍历儿子的过程中找到儿子的兄弟节点，并赋值成同一个parent，然后进行append
appendAllChildren = function(
  parent: Instance,
  workInProgress: Fiber,
  needsVisibilityToggle: boolean,
  isHidden: boolean,
) {
  // We only have the top Fiber that was created but we need recurse down its
  // children to find all the terminal nodes.
  let node = workInProgress.child;
  while (node !== null) {
    if (node.tag === HostComponent || node.tag === HostText) {
      appendInitialChild(parent, node.stateNode);
    } else if (node.tag === FundamentalComponent) {
      appendInitialChild(parent, node.stateNode.instance);
    } else if (node.tag === HostPortal) {
      // If we have a portal child, then we don't want to traverse
      // down its children. Instead, we'll get insertions from each child in
      // the portal directly.
    } else if (node.child !== null) {
      node.child.return = node;
      node = node.child;
      continue;
    }
    if (node === workInProgress) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
};

// react-dom/src/client/ReactDOMHostConfig.js
export function appendInitialChild(
  parentInstance: Instance,
  child: Instance | TextInstance,
): void {
  // 这里调用的就是html的方法了
  parentInstance.appendChild(child);
}
```