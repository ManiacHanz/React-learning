
不同的人看源码有不同的方法，我这里打算从下面这句大家基本都不写的代码入手

```js
ReactDOM.render(<App />, document.querySelector('#root'))
```

会找到`react-dom/src/client`文件夹下面的`ReactDOM.js`文件
在里面我们能看到许多眼熟的方法


```js
// 只放了一些眼熟常用的代码
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
    invariant(
      isValidContainer(container),
      'Target container is not a DOM element.',
    );
    if (__DEV__) {
      warningWithoutStack(
        !container._reactHasBeenPassedToCreateRootDEV,
        'You are calling ReactDOM.render() on a container that was previously ' +
          'passed to ReactDOM.%s(). This is not supported. ' +
          'Did you mean to call root.render(element)?',
        enableStableConcurrentModeAPIs ? 'createRoot' : 'unstable_createRoot',
      );
    }
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  },

  unmountComponentAtNode(container: DOMContainer) {
  },

  flushSync: flushSync,
};
```