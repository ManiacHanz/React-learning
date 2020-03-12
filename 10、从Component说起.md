
前面走完了ReactDOM.render的主流程，一部分react的核心原理其实我们已经了解了。但是还有很大一部分东西我们没有深入进去，比如：一般组件怎么update，expirationTime的权重怎么分级的，在调和阶段怎么区分placement或者update? 等等。

所以这边我换了个思路，重新从 `class Foo extends React.Component{}`开始，探究`React.Component`做了什么，如果这一系列走完，计划可以顺藤摸瓜看看其他类型的component: functionalComponent, Portal, SuspenseComponent的区别与效率等等等等

其实在`ReactBaseClasses.js`里的代码很简单，`Component`也不是什么神秘的东西，一个简单的构造函数，`PureComponent`一个简单的继承。

唯一我们应该看的是`setState`和`forceUpdate`，而看着两个需要看到传入的updater是什么，于是要去找`new Component`的过程，但是全局搜索这个并不能找到，所以找下`Component(`，发现其实答案在`beginWork`里根据`workInProgress.tag`去分类型进行调和的过程中处理的，这个所以有关调和的部分我们都放在下一章讲

```js
// react/src/ReactBaseClasses.js

/**
 * Base class helpers for the updating state of a component.
 */
// 基础的ClassComponent
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

Component.prototype.isReactComponent = {};

/**
 * Sets a subset of the state. Always use this to mutate
 * state. You should treat `this.state` as immutable.
 *
 * There is no guarantee that `this.state` will be immediately updated, so
 * accessing `this.state` after calling this method may return the old value.
 *
 * There is no guarantee that calls to `setState` will run synchronously,
 * as they may eventually be batched together.  You can provide an optional
 * callback that will be executed when the call to setState is actually
 * completed.
 *
 * When a function is provided to setState, it will be called at some point in
 * the future (not synchronously). It will be called with the up to date
 * component arguments (state, props, context). These values can be different
 * from this.* because your function may be called after receiveProps but before
 * shouldComponentUpdate, and this new state, props, and context will not yet be
 * assigned to this.
 *
 * @param {object|function} partialState Next partial state or function to
 *        produce next partial state to be merged with current state.
 * @param {?function} callback Called after state is updated.
 * @final
 * @protected
 */
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};

/**
 * Forces an update. This should only be invoked when it is known with
 * certainty that we are **not** in a DOM transaction.
 *
 * You may want to call this when you know that some deeper aspect of the
 * component's state has changed but `setState` was not called.
 *
 * This will not invoke `shouldComponentUpdate`, but it will invoke
 * `componentWillUpdate` and `componentDidUpdate`.
 *
 * @param {?function} callback Called after update is complete.
 * @final
 * @protected
 */
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};

function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;

/**
 * Convenience component with default shallow equality check for sCU.
 */
function PureComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;

export {Component, PureComponent};

```