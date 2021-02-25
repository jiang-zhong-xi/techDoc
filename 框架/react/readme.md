### 前言

react 版本是v16.6，react v16提出了fiber，但是react fiber最重要的内容time slicing还没有正式使用，所以可以使用了再补充。

### 正文

#### fiber

fiber做了什么？fiber提升了react的性能。

fiber是什么？fiber让主线程和react更高效，主线程是单线程而且需要完成脚本执行、布局、绘制等过程，而react面临一些频繁更新、计算密集的场景，通过fiber的批量化和碎片化让react和主线程更好的配合。

fiber的主要数据结构？return指向父fiber、child指向子fiber、sibling指向兄弟fiber、stateNode指向实例或者DOM的引用

从ReactDOM.render出发，到最后界面渲染完成，render内部主要经历了render阶段、commit阶段。

#### render阶段

从HostRoot节点开始，这个节点对应着根DOM，然后开始workLoop，每次workLoop都有一个work-in-progress树的节点，该节点可能是新建的也可能是从current树复用的，这涉及到了diff算法，对于需要的改变，都存储再effect链表和work-in-progress里，最终都会合并给父节点，每完成一个workLoop对于异步的work都会判断下主线程是否有需要做的其它任务，如果有，则暂停当前任务，通过requestIdleCallback等主线程有闲置时间了再继续执行。

#### commit阶段

render阶段把需要改变的节点都存储在了sideEffect链表，最终这些链表都汇总到了HostRoot的effect属性，然后遍历effect属性对DOM做出操作，整个commit阶段不能被打断。

##### diff

workLoop过程中，每次重新生成虚拟DOM时会和current树的对应节点做比较，如果新的当前节点是单元素，则从current树对应的节点中和当前节点比较，如果key和type（元素类型）都相同说明可以复用，则把current树中的节点克隆给当前work-in-progress树，然后在下次workLoop中更新props和state即可，如果key或者type不同则根据当前元素新建fiber；如果当前节点是列表则是以下逻辑：

1. 首先匹配相同索引号的元素和旧的fiber，如果key不同则直接退出匹配循环，如果key相同，比较type是否相同，相同则复用，不同则新建。

2. 如果新的元素遍历完毕了则直接退出reconcileChildrenArray。

3. 如果oldFiber遍历完了，则把剩下的新元素对应的fiber都新建。

4. 如果oldFiber还有，新元素也没遍历完，则通过map形式进行新元素和旧fiber是否可重用的匹配。

  这里有个关键点是lastPlacedIndex，这个值始终保持新元素的fiber如果是复用的旧fiber的最大的index，这样就能实现复     用的节点保持不动，提高性能。

 再就是如果新元素和旧fiber是完全对应的（key，index，element）则在第一轮后直接旧匹配完了。

  就算最后新元素和旧节点都有，那么也通过map实现高效的匹配。

#### setState

三种调用setState的场景，生命周期中、事件系统、延迟函数中。

生命周期中如componentWillMount，setState调用后，更新链表会添加一个更新，而更新链表会在生命周期函数执行完毕后执行更新，所以在本次任务的这个步骤后就能把state更新。

事件系统内中如onClick事件，setState调用后，由于是批量更新，所以等到当前事件的函数体执行完毕，然后在下一次的任务会对当前节点对应的fiber以及子fiber检查是否有更新。

延迟函数中如setTimeout，此时react没有执行中的任务，所以setState调用后，立马就可以得到更新后的state。

#### 事件

1. 首先是调用组件的render函数，把事件添加到虚拟DOM的属性中。
2. 根据sideEffect创建真实DOM时需要对DOM的属性初始化，此时会把DOM对应的事件以及处理炳添加进去，这里不是添加组件的事件处理柄，而是添加一个顶层组件（body）对该事件的监听器，这是一种委托模式，减少内存损耗。
3. 在DOM上触发事件时，监听器得到触发，获取当前事件的事件源、参数、类型。
4. 从事件源、参数、类型中获取事件源DOM对应的fiber、合并react事件参数。
5. 通过fiber的return属性，找到父代fiber中包含该事件处理柄的fiber，把这个fiber和对应的处理柄存储起来，做捕获或者冒泡用。
6. 通过自定义事件来触发事件处理柄，自定义事件的好处是可以捕获错误，不影响后面的执行。

### 小技巧

1. 单向链表，通过存储指针来实现元素之间的连接，相比于数组有如下特点：

   数组是静态内存分配并且内存是连续的，单向链表是动态内存分配并且内存可以是不连续的；查找元素时数组的时间复杂度为O(1)（通过[index]，因为有寻址公式，所以很快）、O(n)（通过遍历匹配，不连续的只能通过指向一个个匹配）,而单向链表是O(n)；插入或者删除元素时数组是O(n)（因为是连续内存所以要移动的元素多，时间复杂度也高），而单向链表是O(1)；

   ```javascript
   // queue的连接方式，通过firstUpdate的next连接为一个列表链，而列表链中的最后一个指向lastUpdate
   // 而lastUpdate的next是没有的，而最后一个的next的是null
   function appendUpdateToQueue<State>(
     queue: UpdateQueue<State>,
     update: Update<State>,
   ) {
     // Append the update to the end of the list.
     // 如果当前queue是空的,那么第一次和最后一个的指针都指向update
     if (queue.lastUpdate === null) {
       // Queue is empty
       queue.firstUpdate = queue.lastUpdate = update;
     } else {
       // 非空，把最后一个lastUpdate的next指向update, 而把update赋值为最后一个lastUpdate
       queue.lastUpdate.next = update;
       queue.lastUpdate = update;
     }
   }
   ```

2. js 取整

   ```
   num | 0
   ```

   另外取整的方式由parseInt()，向上取整Math.ceil()，向下取整Math.floor()

   自定义精度的取整

   ```javascript
   function ceiling(num: number, precision: number): number {
     return (((num / precision) | 0) + 1) * precision;
   }
   ```

   

### 面试题

#### React 中 keys 的作用

key是用来标识react对应的JSX DOM和fiber的。

特点1：在如果组件时列表项目时才起作用，如果key不同react会认为是不同的元素，key相同，再去比较type是否一样。

特点2：key这是在兄弟间唯一，因为diff算法只会在同级节点间比较。

key标识了JSX DOM后react就能对已有节点实现复用，其实相当于开发者告诉react：我这个组件的“主键”就是这个key，只要key不变，那么组件内容直接复用即可。

#### setState后发生了什么

setState调用后，首先取出实例对应的fiber，然后再fiber上的更新队列上添加本次更新，然后把这个fiber对应的root安排进rootFiber的待更新链表里，如果是通过react系统事件触发的，那么会设置批量标识会在事件执行完毕后一起更新队列，此时state是异步的，而如果是通过setTimeout回调或者网络请求回调触发，此时是同步的，也就是执行一次setState触发一次fiber渲染。

关于更新队列updateQueue？

#### react生命周期

“render阶段”（解析JSX DOM 为fiber的过程）触发constructor（初始化时），然后是getDerivedStateFromProps（根据props获取state，初始化和更新都有），然后是shouldComponentUpdate（更新时），render（初始化和更新都有），getSnapshotBeforeUpdate（更新时），componentDidMount（初始化），componentDidUpdate（更新时），componentWillUnmounted（最后阶段）。

#### shouldComponentUpdate

首先检查是否实例上是否声明了shouldComponentUpdate，如果声明了就用shouldComponentUpdate的执行结果，然后检查实例原型上是否有pureCompoent，如果有则进行“浅比较”，如果是对象则比较属性值，如果不是对象则通过Object.is比较，如果结果是true则调用render。

#### 高阶组件

高阶组件是接受组件参数返回新组件的函数，目的是实现逻辑的复用，普通组件是将props和state转为UI，而高阶组件是将组件转为另一个组件。

#### super的作用

构造函数中调用super相当于调用父类的构造函数。

#### react Hooks

动机：1. 之前组件间复用逻辑困难，props或者高阶组件都会形成嵌套低于。2. 组件逻辑复杂，componentDidMount中充斥大量逻辑。2. class的学习成本高。

从一个页面（权益包订单列表）的重构分析hooks，hooks重构后确实代码量少了，因为state和setState的使用确实简洁了，但是useState被调用了很多次，这个看着有点难受；useEffect异步执行的，所以比生命周期函数性能更好，对于某些需要先订阅后删除的场景支持性更好；关于自定义hooks实现代码复用确实比高阶组件好多了，没那么多嵌套，而是做了层提取就能实现；

useCallback 和 useMemo像是vue中的computed的实现，只有依赖项更新时才重新计算；useRef是react class组件中ref的实现。

#### react性能优化

1. 开发环境使用开发版本的react，生产环境使用生产环境的react，因为react生成环境会删去很多警告类的代码。
2. react内部本身已经做了很多优化，比如对DOM的重用，因为操作DOM很昂贵，react会对新生成的元素和已有的fiber做比较，如果key和type相同直接“克隆”，不过我们还可以通过componentShouldUpdate或者pureComponent进一步优化性能。
3. 因为pureComponent属于浅比较，如果我们确实需要更新state，可以使用concat、spread语法、Object.Assign，改变对象的引用。
4. 对于长列表，可以使用“列表虚拟化”技术，如react-windowing，对列表项懒加载。