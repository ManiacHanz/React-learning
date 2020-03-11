以ReactDOM.render来说
总体大概步骤：

start ->  init  ->  updateQueue  ->  schedule  ->  render  ->  commit

init 处理 FiberRoot 及 rootFiber 的初始化工作，以及挂在到root._reactRootContainer._internalRoot

updateQueue阶段
1. 这里会计算expirationTime，当然如果是ReactDOM.render进来，会把权重设置成最高，变成Sync
2. 会初始化update对象，也就是rootFiber的update对象
3. 会初始化updateQueue链表，然后挂载到fiber.updateQueue属性上
4. 会通过enqueueUpdate，把updateQueue上的update形成一个链表：这里会包含处理的动作的update，以及捕获问题的capturedUpdate
5. 进入调度

schedule阶段
1. 会通过markUpdateTimeFromFiberToRoot函数，计算出优先级比较高的expirationTime
2. 执行renderRoot，并且通过prepareFreshStack重置重要的变量*workInProgress*以及*workInProgressRoot*
3. 进入workLoop循环

react进入workLoop以后的调度流程：
1. 从workLoop进入performUnitOfWork(workInProgress)，这时的WIP是当前文件的全局wIP, 是上一部根据fiberNode创建的wIp；
2. performUnitOfWork会首先进入beginWork方法开始工作
3. beginWork是通过传入的这个workInProgress.tag, 分发对于不同的Component的处理方法, 遍历改变update上的effect。然后返回workInProgress.child到performUnitOfWork
4. 如果此时如果返回的workInProgress.child !== null，就回到performUnitOfWork方法里，再次进入beginWork(workInProgress.child)逻辑
5. 如果某次beginWork返回的workInProgress.child === null，会把这个wIP传入到completeUnitOfWork方法里进行另一个操作
6. 在completeUnitOfWork里首先会有一个completeWork, 这里如果遇到suspenseComponent, 并且没有执行完，会直接返回这个wIP, 回到performUnitOfWork里
7. completeUnitOfWork如果没有suspenseComponent，会把当前wIP的efffect链表接到的return的effect链表上, 然后找到wIP的sibling.
8. 如果sibling !== null 会把sibling返回performUnitOfWork，继续走beginWOrk循环
9. 如果sibling === null 会找到当前unit.returnFiber，也就是它的父节点，不跳出，直接继续completeUnitOfWork流程，知道unit.returnFiber === null 则跳出
10. 完成work（renderContext）阶段，进入commit阶段 

commit 阶段

1. commitRoot 
2. 根据fiber.tag执行遍历的更新： 更新前，更新，更新后。分别代表对应的生命周期
3. 通过getPublicRootInstance返回生成的实例