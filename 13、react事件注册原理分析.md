
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
        if (!newProps) {
          invariant(
            workInProgress.stateNode !== null,
            'We must have new props for new mounts. This error is likely ' +
              'caused by a bug in React. Please file an issue.',
          );
          // This can happen when we abort work.
          break;
        }

        const currentHostContext = getHostContext();
        // TODO: Move createInstance to beginWork and keep it on a context
        // "stack" as the parent. Then append children as we go in beginWork
        // or completeWork depending on we want to add then top->down or
        // bottom->up. Top->down is faster in IE11.
        let wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          // TODO: Move this and createInstance step into the beginPhase
          // to consolidate.
          if (
            prepareToHydrateHostInstance(
              workInProgress,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            // If changes to the hydrated node needs to be applied at the
            // commit-phase we mark this as such.
            markUpdate(workInProgress);
          }
        } else {
          let instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );

          appendAllChildren(instance, workInProgress, false, false);

          if (enableFlareAPI) {
            const listeners = newProps.listeners;
            if (listeners != null) {
              updateEventListeners(
                listeners,
                instance,
                rootContainerInstance,
                workInProgress,
              );
            }
          }

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