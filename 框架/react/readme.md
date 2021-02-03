### 前言

react 版本是v16.6，react v16提出了fiber，但是react fiber最重要的内容time slicing还没有正式使用，所以可以使用了再补充。

### 正文

从ReactDOM.render出发，到最后界面渲染完成，render内部主要经历了render阶段、commit阶段。

#### render阶段

render

#### commit阶段

##### diff

#### setState

#### 事件

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