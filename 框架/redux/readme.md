#### Provider

作为APP的顶层容器，从react-redux中获取，接受一个store作为属性。

#### createStore

从redux中获取，创建store对象，作为Provider的属性。

#### reducer

其含义类似与Array.prototype.reducer，把新state和之前的state合并，这里是处理state的地方，由action来通知reducer进行何种操作。

#### actions

用来通知reducer处理state，其中的type属性和reducer中type属性可以单独拎到一个文件中，payload属性承载新的state。

#### dispatch

派发actions给reducer，使用方式就是调用dispatch方法，以某个actions做为参数，就会触发对应的方法。

#### connect

作为store和UI库的中间件，一种高阶组件的实现，包括两个参数mapStateToProps，mapDispatchToProps，connect函数的返回值也是一个函数，该函数接受一个组件作为参数。

1. mapStateToProps，必须是函数，该函数接受两个参数state和ownProps，state是全部的state，ownProps是传给组件的props，返回值是部分state，ownProps不是必须的，当我们的state需要ownProps时才有必要。
2. mapDispatchToProps，接受一个函数或者一个对象，如果是函数该函数接受一个dispatch参数，返回一个对象，对象中包含的都是dispatch action的方法，在方法里可以做一些处理；如果是一个对象，该对象直接就是dispatch的action。

被绑定的组件可以选择性的把store.state或者某action作为props，一旦返回的“订阅”的state改变了则触发组件重渲染。

#### 为什么使用react redux

redux是一个单独的库，可以用在很多框架中，而react redux的作用就是把react UI和redux进行绑定。react redux完成了状态订阅、更新状态的任务，通过Provider和mapStateToProps订阅初始状态和更新后的状态，通过mapDispatch触发state更新。

#### react redux中间件

redux thunk，用来处理异步的dispatch，因为react redux只能处理同步的流程，但是有些场景需要异步操作，比如从服务器获取数据，而redux作者提供了thunk中间件，实现原理很简单，如果dispatch的是一个函数，那么执行这个函数并把dispatch传过去。