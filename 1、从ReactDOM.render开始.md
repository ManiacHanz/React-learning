
不同的人看源码有不同的方法，我这里打算从下面这句代码入手

```js
ReactDOM.render(<App />, document.querySelector('#root'))
```

会找到`react-dom/src/client`文件夹下面的`ReactDOM.js`文件
在里面我们能看到许多眼熟的方法。能看到，`render`方法其实是调用的`legacyRenderSubtreeIntoContainer`方法

```js
// 只放了一些眼熟常用的代码 ，断言和dev的都删了
const ReactDOM: Object = {
  createPortal,

  findDOMNode(
    componentOrElement: Element | ?React$Component<any, any>,
  ): null | Element | Text {
  },

  hydrate(element: React$Node, container: DOMContainer, callback: ?Function) {
  },

  /**
   * 这里render方法其实是根据传进去的container，初始化了root对象（这个是带_internalRoot的这个）
   * 然后在上面挂再rootFiber对象作为整个fiber树
   * 而之后又把这个rootFiber上的stateNode，即指回来的node节点返回来
   * @param {<APP/>} element
   * @param {('#root')} container
   * @param {*} callback
   */
  render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  },
	// 可以卸载指定的component,应该属于比较好用但是用的比较少的
  unmountComponentAtNode(container: DOMContainer) {
  },

  flushSync: flushSync,
};
```

我们在同一文件下能找到`legacyRenderSubtreeIntoContainer`方法

这个方法分两步来看，第一步是初始化root/fiberRoot等，第二步是执行updateContainer

```js
/**
 * 在初始化调用ReactDom.render(<App />, document.querySelector('#root'))时
 * 会走到这个地方， parentComponent会为null, forceHydrate会为false
 * @param {*} parentComponent
 * @param {*} children
 * @param {*} container
 * @param {*} forceHydrate
 * @param {*} callback
 */
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  // TODO: Without `any` type, Flow says "Property cannot be accessed on any
  // member of intersection type." Whyyyyyy.
  let root: _ReactSyncRoot = (container._reactRootContainer: any);
  let fiberRoot;
  /**
   * 初始化的时候 在container上肯定找不到_reactRootContainer属性，所以会走上面部分
   */
  if (!root) {
    // Initial mount
    /**
     * 初始化就会挂在这个属性
     * 返回的是this._interRoot的属性为fiberRoot的对象
     * fiberRoot.current === rootFiber
     * rootFiber.stateNode === fiberRoot
     * fiberRoot是个root对象  rootFiber是个fiber对象
     * 这里赋值的root又是另外一个对象，是给document.querySelector('root')这个节点赋值的_reactRootContainer属性的对象
     * 而可以通过root._internalRoot就能访问到这个FiberRoot对象
     */
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    /** 在这里挂载了fiberRoot */
    fiberRoot = root._internalRoot;
		/** 
		  部分情景会传入第三个参数callback
			比如 只用redux不用react-redux的时候，可以传入第三个callback
					保证在redux里面数据更新以后，视图能够随之更新
					（但其实react-redux做的不只这些）
		 */
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    /**
     * unbatchedUpdates 代表不是批量更新，这里和包括setState在内的整个调度和更新流程有关
     * 这里先理解成相当于这里直接执行了updateContainer方法
     * 先直接看里面的updateContainer方法
     */
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    /** root已经存在表示非第一次，后期来看 */
  }
  
  return getPublicRootInstance(fiberRoot);
}
```

先看更简单的`getPublicRootInstance`
注意：这里由于上面执行了updateContainer方法，可以先理解成已经把整个树遍历完并且附进去了，且html已经加载完成了

```js
// react-reconciler/src/ReactFiberReconciler.js
export function getPublicRootInstance(
  container: OpaqueRoot,
): React$Component<any, any> | PublicInstance | null {
	// 拿到 FiberRoot 的 fiber
  const containerFiber = container.current;
	// 在updateContainer的过程时，已经把child遍历上去了，所以这里不会是null
	// 除非是组件本身是null
  if (!containerFiber.child) {
    return null;
  }
	// 返回.child.stateNode 也就是代表APP组件的这个stateNode
  switch (containerFiber.child.tag) {
    case HostComponent:
      return getPublicInstance(containerFiber.child.stateNode);
    default:
      return containerFiber.child.stateNode;
  }
}

// shared/ReactWorkTags.js
export const FunctionComponent = 0;
export const ClassComponent = 1;
export const IndeterminateComponent = 2; // Before we know whether it is function or class
export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
export const HostComponent = 5;
export const HostText = 6;
```

`updateContainer`核心，包括从计算`expirationTime`，然后开始初始化`update`和`updateQueue`，然后进入`schedule`的过程。可以说整个react的核心就在这里了，从这里得慢慢来

```js
// react-reconciler/src/ReactFiberReconciler.js

// updateContainer(<App />, fiberRoot, null, undefined);

export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  // 通过current拿到RootFiber
  const current = container.current;
  /**
   * 获取当前时间，其实是一个相对时间
	 * 主要用来计算expirationTime，而这个expirationTime是个重要的用来计算更新优先级的指标
   * 从程序开始，这个currentTime会越来越小
   */
  const currentTime = requestCurrentTime();
  // suspenseConfig 和 lazy 有关， 暂时略过
  const suspenseConfig = requestCurrentSuspenseConfig();
  /**
   * 通过currentTime去拿expirationTime，这就是比较重要的计算时间的方法
   * 通过ReactDOM.render走进来的时候 这里会是 Sync 最大的值
	 * 表示优先级最大，需要立即调度执行
   */
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig,
  );
	// 这里 会返回一个 通过计算时间后的container
	// 当然在ReactDOM.render进来并没有用到return 的内容
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    suspenseConfig,
    callback,
  );
}
```

下一章我们会先看看关于`expirationTime`的相关计算

