

### 1. 前言

本次React Router的源码分析是针对[v3](https://github.com/ReactTraining/react-router/tree/v3/docs)版的，最新版的是v5了，下次有时间再对新版本的分析，源码调试的代码的[github](https://github.com/jiang-zhong-xi/react-router-source-code)地址,nodeJs版本是v12.18.4。

### 2. 正文

React Router好久之前就开始用了，这次想弄明白的一是“项目启动时React Router内部发生了什么”，再就是“路由改变时React Router内部发生了什么”。

##### 2.1 项目启动

代码地址已放在前言中，克隆下来后安装依赖，然后直接“npm start”，访问“http://localhost:8080/”。

而路由配置代码如下：

```javascript
<Router history={browserHistory}>
    <Route path="/" component={App}>
      <Route path="/repos" component={Repos} onEnter={(nextState, replace) => {
        console.warn('nextState', nextState)
      }}>
        <Route path="/repos/:userName/:repoName" component={Repo}/>
        <Route path="about" component={About}>
        </Route>
      </Route>
      <Route path="**" component={All}>
      </Route>
    </Route>
  </Router>
```

从“http://localhost:8080/”，然后经过如上代码，最后展示如上界面，下面开启如上代码的分析旅程！

##### Router

Router是路由配置的根组件，这里先分析history的值browserHistory和hashHistory。

###### browserHistory

对应的模块在“\src\react-router-3.0.0\modules\browserHistory.js”，browserHistory依赖浏览器的history API。

```javascript
import createBrowserHistory from 'history/lib/createBrowserHistory'
import createRouterHistory from './createRouterHistory'

export default createRouterHistory(createBrowserHistory)
```

history是另外一个单独的依赖包，而createRouterHistory的作用是对history再次包装，这里用到了*装饰者模式* ，包装的两个核心函数useBasename和useQueries，先分析createBrowserHistory回来再看这俩函数的作用。

把createBrowserHistory拆分为三个主要部分BrowserProtocol、createHistory、相关listener。

1. BrowserProtocol

   ```javascript
   const PopStateEvent = 'popstate'
   /**
    * 1. state同步到sessionStorage
    * 2. 把路径对象拼接为字符串然后传给h5的API
    * 3. 把h5 location的值取出来封装为自己的对象
    * 4. 开启监听
   */
   const _createLocation = (historyState) => {
     const key = historyState && historyState.key
   
     return createLocation({
       pathname: window.location.pathname, // 指主机名和参数之间的部分
       search: window.location.search, // 参数部分
       hash: window.location.hash, // #以及之后的部分  https://www.cnblogs.com/canger/p/7595641.html
       state: (key ? readState(key) : undefined) // readState从sessionStorage中读取Key对应的值
     }, undefined, key)
   }
   // 创建location对象（pathname、search、hash、state）
   export const getCurrentLocation = () => {
     let historyState
     try {
       historyState = window.history.state || {}
     } catch (error) {
       // IE 11 sometimes throws when accessing window.history.state
       // See https://github.com/mjackson/history/pull/289
       historyState = {}
     }
   
     return _createLocation(historyState)
   }
   // 把用户选择的确定还是取消，然后作为参数传给callback
   export const getUserConfirmation = (message, callback) =>
     callback(window.confirm(message))
   /**
    * startListener函数执行后触发开启监听，startListener函数执行取消监听，这样看起来结构更紧密
    * 给window的popstate事件绑定事件处理柄，并把处理后的location传给listener
   */
   export const startListener = (listener) => {
     const handlePopState = (event) => {
       if (event.state !== undefined) // Ignore extraneous popstate events in WebKit
         listener(_createLocation(event.state))
     }
   // 注意：调用history.pushState()或者history.replaceState()不会触发popstate事件. popstate事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者在JavaScript中调用history.back()、history.forward()、history.go()方法)，此外，a 标签的锚点也会触发该事件.
     addEventListener(window, PopStateEvent, handlePopState)
   
     return () =>
       removeEventListener(window, PopStateEvent, handlePopState)
   }
   // 这里会把state存储到sessionStorage，这是React Router的一种传参方式
   const updateLocation = (location, updateState) => {
     // key值的生成逻辑在./createHistory中，state是调用改变路由API时传递的，这样就实现了隐式传递参数
     // 并且state是保存在window.location.state中，state值的变化会触发PopStateEvent的监听器
     const { state, key } = location
   
     if (state !== undefined)
       // state存储到sessionStorage
       saveState(key, state)
       // 同步到window.location.state中, createPath 把location中的basename、pathname、search、hash拼接在一起
     updateState({ key }, createPath(location))
   }
   export const pushLocation = (location) =>
     updateLocation(location, (state, path) =>
       // pushState浏览器会话历史栈中添加一个地址，和location重定位不同的是即使是一样的pathname pushState也会添加，但location只能是给相同的pathname添加不同hash
       window.history.pushState(state, null, path)
     )
   export const replaceLocation = (location) =>
     updateLocation(location, (state, path) =>
       //
       window.history.replaceState(state, null, path)
     )
   // 直接调用原生的go函数
   export const go = (n) => {
     if (n)
       window.history.go(n)
   }
   ```

   可以看到BrowserProtocol里的大部分函数是对H5 history的包装，大部分已加注释。

2. createHistory

   ```javascript
   /**
    * 1. 保持历史会话栈和allKeys同步，在用户取消“window.confirm”时利用allKeys恢复之前的路径
    * 2. 添加listener和beforeListener，路由更新（包括通过API或者a或者地址栏底子好）时执行对应的beforeListener和listener
   */
   const createHistory = (options = {}) => {
     const {
       getCurrentLocation,
       getUserConfirmation,
       pushLocation,
       replaceLocation,
       go,
       keyLength
     } = options
   
     let currentLocation
     let pendingLocation
     let beforeListeners = []
     let listeners = []
     let allKeys = []
     // 当前key在对应的地址在历史栈中的位置
     const getCurrentIndex = () => {
       if (pendingLocation && pendingLocation.action === POP)
         return allKeys.indexOf(pendingLocation.key)
   
       if (currentLocation)
         return allKeys.indexOf(currentLocation.key)
   
       return -1
     }
     // 更新allKeys（代表页面栈的所有路径）里的key，执行所有listener
     const updateLocation = (nextLocation) => {
       currentLocation = nextLocation
   
       const currentIndex = getCurrentIndex()
   
       if (currentLocation.action === PUSH) {
         allKeys = [ ...allKeys.slice(0, currentIndex + 1), currentLocation.key ]
       } else if (currentLocation.action === REPLACE) {
         allKeys[currentIndex] = currentLocation.key
       }
   
       listeners.forEach(listener => listener(currentLocation))
     }
     /**
      * 通过闭包，在函数体里添加一个listener，在函数执行后返回的函数体里把当前listener去除
     */
     const listenBefore = (listener) => {
       beforeListeners.push(listener)
   
       return () =>
         beforeListeners = beforeListeners.filter(item => item !== listener)
     }
     // 同上
     const listen = (listener) => {
       listeners.push(listener)
   
       return () =>
         listeners = listeners.filter(item => item !== listener)
     }
     // 执行所有的beforeListeners，如果需要执行getUserConfirmation
     const confirmTransitionTo = (location, callback) => {
       loopAsync(
         beforeListeners.length,
         (index, next, done) => {
           runTransitionHook(beforeListeners[index], location, (result) =>
             result != null ? done(result) : next()
           )
         },
         (message) => {
           if (getUserConfirmation && typeof message === 'string') {
             getUserConfirmation(message, (ok) => callback(ok !== false))
           } else {
             callback(message !== false)
           }
         }
       )
     }
     // 完成路由的过渡，执行beforeListener，如果存在则根据用户弹框判断是取消还是继续完成接下来的路由
     const transitionTo = (nextLocation) => {
       // 接下来的路由相等则直接停止接下来的操作
       if (
         (currentLocation && locationsAreEqual(currentLocation, nextLocation)) ||
         (pendingLocation && locationsAreEqual(pendingLocation, nextLocation))
       )
         return // Nothing to do
   
       pendingLocation = nextLocation
       // 是否需要弹出确认弹框
       confirmTransitionTo(nextLocation, (ok) => {
         if (pendingLocation !== nextLocation)
           return // Transition was interrupted during confirmation
   
         pendingLocation = null
   
         if (ok) {// 用户点击了确认那么调用updateLocation（执行listener、更新AllKeys）,调用history对应的API
           // Treat PUSH to same path like REPLACE to be consistent with browsers
           if (nextLocation.action === PUSH) {
             const prevPath = createPath(currentLocation)
             const nextPath = createPath(nextLocation)
   
             if (nextPath === prevPath && statesAreEqual(currentLocation.state, nextLocation.state))
               nextLocation.action = REPLACE
           }
   
           if (nextLocation.action === POP) {
             updateLocation(nextLocation)
           } else if (nextLocation.action === PUSH) {
             if (pushLocation(nextLocation) !== false)
               updateLocation(nextLocation)
           } else if (nextLocation.action === REPLACE) {
             if (replaceLocation(nextLocation) !== false)
               updateLocation(nextLocation)
           }
         } else if (currentLocation && nextLocation.action === POP) { // 用户取消恢复之的URL
           const prevIndex = allKeys.indexOf(currentLocation.key)
           const nextIndex = allKeys.indexOf(nextLocation.key)
   
           if (prevIndex !== -1 && nextIndex !== -1)
             go(prevIndex - nextIndex) // Restore the URL
         }
       })
     }
     // input调用API传的地址，添加动作类型PUSH
     const push = (input) =>
       transitionTo(createLocation(input, PUSH))
     // input调用API传的地址，添加动作类型REPLACE
     const replace = (input) =>
       transitionTo(createLocation(input, REPLACE))
     // 退回历史栈中上一页
     const goBack = () =>
       go(-1)
     // 前往历史栈中下一页
     const goForward = () =>
       go(1)
   
     const createKey = () =>
       /**
        * Math.random()生成一个0-1之间的随机数
        * toString(36)将随机数转为36机制然后转为字符串，36进制包括了10个数字26个英文字母
        * substr(2, keyLength || 6)，从第二位开始，一共6位
       */
       Math.random().toString(36).substr(2, keyLength || 6)
     // 把location中的basename、pathname、search、hash拼接在一起
     const createHref = (location) =>
       createPath(location)
     /**
      * 创建location对象包括pathname,search,hash,state,action,key
     */
     const createLocation = (location, action, key = createKey()) =>
       _createLocation(location, action, key)
   
     return {
       getCurrentLocation,
       listenBefore,
       listen,
       transitionTo,
       push,
       replace,
       go,
       goBack,
       goForward,
       createKey,
       createPath,
       createHref,
       createLocation
     }
   }
   ```

3. 其它

   ```javascript
   let listenerCount = 0, stopListener
   
   const startListener = (listener, before) => {
       // 首次执行该函数，则把history的transitionTo设置为BrowserProtocol中window.onpopstatechange的监听者
       // 这样如果无论是通过API去改变路由 还是 通过a链接或者其它方式都会触发transitionTo
       if (++listenerCount === 1)
           stopListener = BrowserProtocol.startListener(
               history.transitionTo
           )
   
       const unlisten = before
       ? history.listenBefore(listener)
       : history.listen(listener)
   
       return () => {
           // 删除
           unlisten()
           // 取消监听
           if (--listenerCount === 0)
               stopListener()
       }
   }
   // 添加监听者,在router的component中的setRouteLeaveHook就是调用的这个函数,setRouteLeaveHook函数返回值作为window.confirm的弹框信息
   const listenBefore = (listener) =>
   startListener(listener, true)
   // 添加监听者,这个接口把react router的路由匹配部分和history监听部分联系到了一起
   const listen = (listener) =>
   startListener(listener, false)
   ```

###### hashHistory

这个模块路径对应的路径是“\src\react-router-3.0.0\modules\hashHistory.js”，hashHistory依赖与window.location.hash，hash属性比较独特，这个路径会展示在url（如：http:\/\/***\/#\/test）中，但是发送http请求时不携带hash部分的值（如：http://\*\*\*/\），浏览器这么干的目的是啥呢？浏览器有把页面收藏到书签栏的功能，如果我们阅读一个很长的文章，每次从书签栏点击后都要从头开始是不是很费劲呢，hash就可以解决这个问题，打开hash地址后滚动到之前保存的位置，当然这需要js代码做处理。在React Router中hashHistory正是用到了这点，另外地址只是hash切换除了不重新请求页面，其余的表现和新的地址是一样的，像改变历史栈。

```javascript
import createHashHistory from 'history/lib/createHashHistory'
import createRouterHistory from './createRouterHistory'
export default createRouterHistory(createHashHistory)
```

无论是browserHistory还是hashHistory都是通过createRouterHistory进行了封装，这里看下封装的两个核心函数useBasename和useQueries的实现逻辑。

```javascript
// 闭包保存createHistory函数
export default function useRouterHistory(createHistory) {
  return function (options) { // 
    const history = useQueries(useBasename(createHistory))(options)
    return history
  }
}
```

useRouterHistory执行后返回一个函数，通过这个函数我们可以自定义一些参数想basename、对象参数解析规则、字符串参数解析规则。使用方法如下

```javascript
baseHistory = useRouterHistory(createHistory)({
    basename: 'custom'
});
```



- useBasename，这个函数的作用是让所有的路径都在某一个basename下，如果我们的项目不能放在服务器根目录下这个配置就能起作用了，配置生成环境下的路由都添加一个basename。

  ```javascript
  // 如果location.pathname中包含basename，则把basename部分赋值给basename属性
  const addBasename = (location) => {
      if (!location)
          return location
  
      if (basename && location.basename == null) {
          if (location.pathname.indexOf(basename) === 0) {
              location.pathname = location.pathname.substring(basename.length)
              location.basename = basename
  
              if (location.pathname === '')
                  location.pathname = '/'
          } else {
              location.basename = ''
          }
      }
  
      return location
  }
  // 根据basename pathname重新拼接pathname
  const prependBasename = (location) => {
      if (!basename)
          return location
  
      const object = typeof location === 'string' ? parsePath(location) : location // 保证location是对象
      const pname = object.pathname
      /** 
         * slice(-1)返回字符串的最后一个字符，这种方式最简洁 
        */
      const normalizedBasename = basename.slice(-1) === '/' ? basename : `${basename}/` // 保证最后一个元素是/
      const normalizedPathname = pname.charAt(0) === '/' ? pname.slice(1) : pname // 保证第一个字符不是/
      const pathname = normalizedBasename + normalizedPathname // 重新拼接
  
      return {
          ...location,
          pathname
      }
  }
  ```

  addBasename是拆分，把basename从pathname中拆分出来，赋值给location.basename，这样我们在获取location你对象的时候比如listener中都能看到localtion.basename；prependBasename是合并，把location.basename和location.pathname拼接到一起，这样我们在调用push、replace后地址栏中是携带basename的url。

- useQueries，这个函数的作用是对对象参数的解析，解析函数我们也可以传入自定义的。

  ```javascript
  const decodeQuery = (location) => {
      if (!location)
          return location
  
      if (location.query == null)
          location.query = parseQueryString(location.search.substring(1))
  
      return location
  }
  
  const encodeQuery = (location, query) => {
      if (query == null)
          return location
  
      const object = typeof location === 'string' ? parsePath(location) : location
      const queryString = stringifyQuery(query)
      const search = queryString ? `?${queryString}` : ''
  
      return {
          ...object,
          search
      }
  }
  
  ```

  很明显这俩函数也是相对的，decodeQuery把&分割的字符串参数解析为对象，这样在获取参数的时候是对象形式，比如自己写的组件this.props.location.query；而encodeQuery就是把参数对象中的键值对拼接为&分割的字符串，这样地址就能呈现我们的参数。

  这样就分析了“history={browserHistory}”的完整历程，接下来分析hashHistory，useRouterHistory部分是一样的，不再赘述。
  
  现在开始分析hashHistory，同样分为三个主要部分HashProtocol、createHistory、其它，其中createHistory、其它是复用的browserHistory所以不再赘述。
  
  1. HashProtocol
  
     ```javascript
     const HashChangeEvent = 'hashchange'
     // 获取地址中#以后的部分
     const getHashPath = () => {
       // 由于浏览器间的兼容，这里没有用window.location.hash
       // We can't use window.location.hash here because it's not
       // consistent across browsers - Firefox will pre-decode it!
       const href = window.location.href
       const index = href.indexOf('#')
       return index === -1 ? '' : href.substring(index + 1)
     }
     // 这样能往页面栈中添加一个地址
     const pushHashPath = (path) =>
       window.location.hash = path
     // 这样实现替换页面栈中地址 如果地址中不包括#那么就开href的开头添加#，如果包括则保持#之前的部分不变然后替换#以及之后的部分
     const replaceHashPath = (path) => {
       const i = window.location.href.indexOf('#')
       // 保持#之前的部分不变，把#以及之后的部分替换
       window.location.replace(
         window.location.href.slice(0, i >= 0 ? i : 0) + '#' + path
       )
     }
     // 确保hash部分是以/开头  初始化时时没有的#，所以通过这个方法添加#以及/
     const ensureSlash = () => {
       const path = getHashPath()
     
       if (isAbsolutePath(path))
         return true
     
       replaceHashPath('/' + path)
     
       return false
     }
     // 复用
     export { getUserConfirmation, go } from './BrowserProtocol'
     // queryKey的作用是作为随即生成的key的“键”，而key用来保存state的，那么query的目的就是把key添加到url中，key的变化会触发HashChangeEvent的监听器
     export const getCurrentLocation = (queryKey) => {
       let path = getHashPath()
       // path中queryKey对应的值读取出来并返回
       const key = getQueryStringValueFromPath(path, queryKey)
     
       let state
       if (key) {
         // queryKey对应的键值对从path中去掉，一次性的
         path = stripQueryStringValueFromPath(path, queryKey)
         // key对应的sessionStorage的值读取
         state = readState(key)
       }
       // 重新拼接为对象path
       const init = parsePath(path)
       // 对象path添加state
       init.state = state
       // 添加action
       return createLocation(init, undefined, key)
     }
     
     let prevLocation
     
     export const startListener = (listener, queryKey) => {
       const handleHashChange = () => {
         
         if (!ensureSlash())
           return // Hash path must always begin with a /
     
         const currentLocation = getCurrentLocation(queryKey)
     
         if (prevLocation && currentLocation.key && prevLocation.key === currentLocation.key)
           return // Ignore extraneous hashchange events
     
         prevLocation = currentLocation
     
         listener(currentLocation)
       }
       // 这个在初始化时很重要，因为我们输入的url可能不带#，而这里则实现了添加#
       ensureSlash()
       addEventListener(window, HashChangeEvent, handleHashChange)
     
       return () =>
         removeEventListener(window, HashChangeEvent, handleHashChange)
     }
     // 保存state，生成location对象
     const updateLocation = (location, queryKey, updateHash) => {
       
       const { state, key } = location
       let path = createPath(location)
       // 把state保存到sessionStorage中
       if (state !== undefined) {
         // 生成包含hash、search、pathname的对象
         path = addQueryStringValueToPath(path, queryKey, key)
         // 保存到sessionStorage
         saveState(key, state)
       }
     
       prevLocation = location
     
       updateHash(path)
     }
     // 更新location，state，然后改变url的hash
     export const pushLocation = (location, queryKey) =>
       updateLocation(location, queryKey, (path) => {
         // hash部分不一样
         if (getHashPath() !== path) {
           pushHashPath(path)
         } else {
           warning(false, 'You cannot PUSH the same path using hash history')
         }
       })
     // 更新location，state，然后替换url的hash
     export const replaceLocation = (location, queryKey) =>
       updateLocation(location, queryKey, (path) => {
         if (getHashPath() !== path)
           replaceHashPath(path)
       })
     ```

###### 组件内部

history属性是Router非常重要的属性，包含了React Router所有底层逻辑，其它的属性像children 、createElement、

render在下面的流程中会提到。React Router库最核心的部分注册路由监听、路由改变后更新组件，而这两部分的“入口”都在Router组件里而对应的实现在createTransitionManager和RouterContext中。

Router组件的componentWillMount

```javascript
componentWillMount() {
    this.transitionManager = this.createTransitionManager() // 监听变化-组件更新的核心逻辑
    this.router = this.createRouterObject(this.state)
    // 监听路由变化，更新完后调用回调函数
    this._unlisten = this.transitionManager.listen((error, state) => {
        if (error) {
            this.handleError(error)
        } else {
            debugger
            // Keep the identity of this.router because of a caveat in ContextUtils:
            // they only work if the object identity is preserved.
            /**
         * 虽然this.router中包含很多属性（history、transitionManager）但是只有location、routes、params、component需要react 追踪变化
         * 这里先把最新的state合并到this.router，然后去更新state，减小了需要追踪的数量-优化策略
        */
            assignRouterState(this.router, state)
            this.setState(state, this.props.onUpdate)
        }
    })
},
```

transitionManager状态是路由和url匹配的部分，router状态最终要作为props传到页面组件中的，这里把路由监听、组件更新结合起来了，先看transitionManager的生成过程。

```javascript
createTransitionManager() {
    const { history } = this.props
    const { routes, children } = this.props
    return createTransitionManager(
        history,
        createRoutes(routes || children)
    )
}
```

history就是browserHistory或者hashHistory对象，而routes|children是router的子组件，即我们配置的路由，哪createRoutes就是根据配置的路由组件生成路由对象的过程。

```javascript
createRoutes(routes) {
  if (isReactChildren(routes)) { // 用的router组件生成的虚拟DOM
    // 迭代遍历出路由对象
    routes = createRoutesFromReactChildren(routes)
  } else if (routes && !Array.isArray(routes)) { // 开发者配置的就是路由对象
    routes = [ routes ]
  }
  return routes
}
```

路由配置我们可以用组件来配置，也可以直接用对象，这里就是两种方式的控制逻辑。

下面来看createTransitionManager的执行过程。

1. 添加监听者

   ```javascript
   const unsubscribe = history.listen(historyListener)
   if (state.location) { // 初始化后路由发生改变
       // Picking up on a matchContext.
       listener(null, state)
   } else { // 路由初始化
       // 输入地址栏地址，js加载完毕后，获取格式化后的地址，传入listener中，完成首次的匹配
       historyListener(history.getCurrentLocation())
   }
   ```

   可以看到首先是把“historyListener”注册进“history.listen”,启动监听路由的改变，路由的改变包括两种，一种是调用push或者replace，还有一种是点击a链接、winodow.location.state，这两种的实现方法不一样，但是最后都会触发historyListener，此时如果是hashHistory，则会给地址添加#。初次渲染不会触发监听者，所以直接调用“historyListener”，而history.getCurrentLocation()会获取当前地址对象。

2. 监听者处理路由对象和当前地址的匹配，以及匹配后的回调

   21. 匹配，这部分完成路由和地址的匹配，匹配的过程中会把路由中的一些特殊字符比如*转为正则表达式

       ```javascript
       function matchRouteDeep(
         route, location, remainingPathname, paramNames, paramValues, callback
       ) {
         let pattern = route.path || '' // 路由组件对应的路径
       
         if (pattern.charAt(0) === '/') { // 如果路由组件的路径是由/开头，那么remainingPathname就赋值当前地址
           remainingPathname = location.pathname
           paramNames = []
           paramValues = []
         }
       
         // Only try to match the path if the route actually has a pattern, and if
         // we're not just searching for potential nested absolute paths.
         if (remainingPathname !== null && pattern) {
           try {
             // 匹配的过程
             const matched = matchPattern(pattern, remainingPathname)
             if (matched) {
               remainingPathname = matched.remainingPathname
               paramNames = [ ...paramNames, ...matched.paramNames ]
               paramValues = [ ...paramValues, ...matched.paramValues ]
             } else {
               remainingPathname = null
             }
           } catch (error) {
             callback(error)
           }
       
           // By assumption, pattern is non-empty here, which is the prerequisite for
           // actually terminating a match.
           // 如果当前地址已经匹配完整了，则取最后一个路由的indexRoute
           if (remainingPathname === '') {
             const match = {
               routes: [ route ],
               params: createParams(paramNames, paramValues)
             }
       
             getIndexRoute(
               route, location, paramNames, paramValues,
               function (error, indexRoute) {
                 if (error) {
                   callback(error)
                 } else {
                   if (Array.isArray(indexRoute)) {
                     warning(
                       indexRoute.every(route => !route.path),
                       'Index routes should not have paths'
                     )
                     match.routes.push(...indexRoute)
                   } else if (indexRoute) {
                     warning(
                       !indexRoute.path,
                       'Index routes should not have paths'
                     )
                     match.routes.push(indexRoute)
                   }
       
                   callback(null, match)
                 }
               }
             )
       
             return
           }
         }
         // 如果地址换没遍历完，并且route对象还有子路由，则继续子路由，如此循环，直到遍历完毕调用回调函数
         if (remainingPathname != null || route.childRoutes) {
           // Either a) this route matched at least some of the path or b)
           // we don't have to load this route's children asynchronously. In
           // either case continue checking for matches in the subtree.
           const onChildRoutes = (error, childRoutes) => {
             if (error) {
               callback(error)
             } else if (childRoutes) {
               // Check the child routes to see if any of them match.
               matchRoutes(childRoutes, location, function (error, match) {
                 if (error) {
                   callback(error)
                 } else if (match) {
                   // A child route matched! Augment the match and pass it up the stack.
                   match.routes.unshift(route)
                   callback(null, match)
                 } else {
                   callback()
                 }
               }, remainingPathname, paramNames, paramValues)
             } else {
               callback()
             }
           }
       
           const result = getChildRoutes(
             route, location, paramNames, paramValues, onChildRoutes
           )
           if (result) {
             onChildRoutes(...result)
           }
         } else {
           callback()
         }
       }
       ```

       matchPattern，将转化好的路由路径和地址栏进行正则匹配，匹配后计算剩余的路径和参数然后存入remainingPathname、paramNames和paramValues

       ```javascript
       function matchPattern(pattern, pathname) {
         // Ensure pattern starts with leading slash for consistency with pathname.
         if (pattern.charAt(0) !== '/') {
           pattern = `/${pattern}`
         }
         /*
           regexpSource 路由路径的对应的正则
           paramNames 路由路径中的参数名称
       
         */
         let { regexpSource, paramNames, tokens } = compilePattern(pattern)
       
         if (pattern.charAt(pattern.length - 1) !== '/') {
           regexpSource += '/?' // Allow optional path separator at end.
         }
       
         // Special-case patterns like '*' for catch-all routes.
         // 添加$（结束字符）标识 匹配到最后一个字符
         if (tokens[tokens.length - 1] === '*') {
           regexpSource += '$'
         }
       
         const match = pathname.match(new RegExp(`^${regexpSource}`, 'i'))
         if (match == null) { // 匹配失败
           return null
         }
       
         const matchedPath = match[0] // 路由路径和当前地址匹配到的子串
         let remainingPathname = pathname.substr(matchedPath.length) // 去除匹配子串后的路径
       
         if (remainingPathname) {
           // Require that the match ends at a path separator, if we didn't match
           // the full path, so any remaining pathname is a new path segment.
           if (matchedPath.charAt(matchedPath.length - 1) !== '/') {
             return null
           }
       
           // If there is a remaining pathname, treat the path separator as part of
           // the remaining pathname for properly continuing the match.
           remainingPathname = `/${remainingPathname}`
         }
       
         return {
           remainingPathname,
           paramNames,
           paramValues: match.slice(1).map(v => v && decodeURIComponent(v))
         }
       }
       ```

       compilePattern，将路径路径中转为正则表达式，尤其是其中的“*”，“**”，“()”，转为一样含义的正则子表达式。

       ```javascript
       function _compilePattern(pattern) {
         let regexpSource = ''
         const paramNames = []
         const tokens = [] // 匹配到子串全部存储到tokens中
         /*
           路由路径的几种配置方法
           :paramName – 参数名称用/, ?, or #间隔多个参数
           () – 可选参数或者路径 Wraps a portion of the URL that is optional. You may escape parentheses if you want to use them in a url using a backslash \
           * – 非贪婪模式匹配 Matches all characters (non-greedy) up to the next character in the pattern, or to the end of the URL if there is none, and creates a splat param
           ** - 贪婪模式匹配 Matches all characters (greedy) until the next /, ?, or # and creates a splat param
         */
        // :([a-zA-Z_$][a-zA-Z0-9_$]*)对应:paramName(abc..xyz_$)   \*\*对应**   *对应* \(\)对应()
         let match, lastIndex = 0, matcher = /:([a-zA-Z_$][a-zA-Z0-9_$]*)|\*\*|\*|\(|\)/g
         // matcher的lastIndex属性当正则由全局g时有效，总是记录本次索引到子串的最后一个位置+1，这里总是表示上次索引的最后一个
         while ((match = matcher.exec(pattern))) { 
           // match.index表示pattern中匹配的子串第一个字符在pattern中的位置
           // lastIndex表示上次匹配中pattern中匹配的子串最后一个字符在pattern中的位置
           // match.index !== lastIndex表示部分子串在matcher正则里，该部分存入tokens
           if (match.index !== lastIndex) {
             tokens.push(pattern.slice(lastIndex, match.index))
             console.log("tokens", tokens)
             // escapeRegExp 特殊字符串中的某些字符加\后可当作元字符去做正则匹配
             regexpSource += escapeRegExp(pattern.slice(lastIndex, match.index))
           }
       
           if (match[1]) { // 匹配路由路径的参数部分
             // ^表示非 ^的意思是非/的一切字符（至少一个）
             regexpSource += '([^/]+)'
             paramNames.push(match[1])
           } else if (match[0] === '**') { // 贪婪模式匹配所有字符
             regexpSource += '(.*)'
             paramNames.push('splat')
           } else if (match[0] === '*') { // 非贪婪模式匹配所有字符
             regexpSource += '(.*?)'
             paramNames.push('splat')
           } else if (match[0] === '(') { // 可选 (?:)? 可选，不捕获分组
             regexpSource += '(?:'
           } else if (match[0] === ')') { // 可选
             regexpSource += ')?'
           }
       
           tokens.push(match[0])
       
           lastIndex = matcher.lastIndex
         }
       
         if (lastIndex !== pattern.length) {
           tokens.push(pattern.slice(lastIndex, pattern.length))
           regexpSource += escapeRegExp(pattern.slice(lastIndex, pattern.length))
         }
       
         return {
           pattern,
           regexpSource,
           paramNames,
           tokens
         }
       }
       ```

       

   22. 匹配后的回调

       匹配后得到的匹配对象如下格式：

       ```javascript
       const match = {
           routes: [ route,route ], // 匹配的所有路由对象
           params: createParams(paramNames, paramValues) // 所有的路由参数
       }
       ```

       传入匹配后的回调函数

       ```javascript
       function (error, nextState) {
           if (error) {
               callback(error)
           } else if (nextState) {
               finishMatch({ ...nextState, location }, callback)
           } else {
               callback()
           }
       }
       ```

       finishMatch，React Router需要我们在路由上添加一些钩子函数，比如onEnter，这些钩子函数的执行就是在finishMatch里。

       ```javascript
       function finishMatch(nextState, callback) {
           // 计算nextState的路由那些是新添的、那些是去掉的、那些是改变的
           const { leaveRoutes, changeRoutes, enterRoutes } = computeChangedRoutes(state, nextState)
           // 运行离开路由的onLeave钩子函数
           runLeaveHooks(leaveRoutes, state)
           // 去掉监听
           // Tear down confirmation hooks for left routes
           leaveRoutes
               .filter(route => enterRoutes.indexOf(route) === -1)
               .forEach(removeListenBeforeHooksForRoute)
           // 运行change和ente钩子函数r
           // change and enter hooks are run in series
           runChangeHooks(changeRoutes, state, nextState, (error, redirectInfo) => {
               if (error || redirectInfo)
                   return handleErrorOrRedirect(error, redirectInfo)
               // 这里有个知识点：我们定义enterHooks时如果是2个参数那么就是同步执行， 如果是3个异步执行
               runEnterHooks(enterRoutes, nextState, finishEnterHooks)
           })
       ```

       finishEnterHooks，在上述步骤中我们完成了路由监听、路由匹配，是不是还缺少一环，对那就是路由对应的组件的渲染，要想渲染组件首先需要由组件，收集组件的过程就发生在finishEnterHooks。

       ```javascript
       function finishEnterHooks(error, redirectInfo) {
           if (error || redirectInfo)
               return handleErrorOrRedirect(error, redirectInfo)
       
           // TODO: Fetch components after state is updated.
           // 遍历nextState中的routes，取出其中的components或者component，或者getComponents或者getComponent
           // 放入components
           getComponents(nextState, function (error, components) {
               if (error) {
                   callback(error)
               } else {
                   // TODO: Make match a pure function and have some other API
                   // for "match and update state".
                   callback(null, null, (
                       state = { ...nextState, components })
                           )
               }
           })
       }
       ```

       match函数的回调，这里可能由于重定向回到开始的路由监听过程，也可能是继续往下执行listener的回调

       ```javascript
       function (error, redirectLocation, nextState) {
           if (error) {
               listener(error)
           } else if (redirectLocation) {
               // 如果在enterHooks中执行了第二个参数replace，那就是执行了重定向，在这里调用history.replace
               history.replace(redirectLocation)
           } else if (nextState) {
               // 如果继续往下执行就到listener函数的执行了
               listener(null, nextState)
           } else {
               warning(
                   false,
                   'Location "%s" did not match any routes',
                   location.pathname + location.search + location.hash
               )
           }
       }
       ```

       listener的回调，这里更新react的state，触发render

       ```javascript
       (error, state) => {
           if (error) {
               this.handleError(error)
           } else {
               // Keep the identity of this.router because of a caveat in ContextUtils:
               // they only work if the object identity is preserved.
               /**
                * 虽然this.router中包含很多属性（history、transitionManager）但是只有location、routes、params、component需要react 追踪变化
                * 这里先把最新的state合并到this.router，然后去更新state，减小了需要追踪的数量-优化策略
               */
               assignRouterState(this.router, state)
               this.setState(state, this.props.onUpdate)
           }
       }
       ```

       react的render

       ```javascript
       render() {
           const { location, routes, params, components } = this.state
           const { createElement, render, ...props } = this.props
           console.log('render', render)
           if (location == null)
             return null // Async match
       
           // Only forward non-Router-specific props to routing context, as those are
           // the only ones that might be custom routing context props.
           Object.keys(Router.propTypes).forEach(propType => delete props[propType])
       
           return render({
             ...props,
             router: this.router,
             location,
             routes,
             params,
             components,
             createElement
           })
         }
       ```

       这个过程中还有一个render，从属性中读取，看下默认值。

       ```javascript
       render(props) {
           return <RouterContext {...props} />
       }
       ```

       既然说是默认值，当然也可以自定义，自定义render的作用就可以进行数据劫持，我们可以改变props甚至改变RouterContext的实现，另外还有一个createElement，这个的默认值定义在RouterContext中。

       ```javascript
       return {
           createElement: React.createElement
       }
       ```

       有了createElement，可以说render是一种总关口的数据劫持，而createElement是对每个组件创建的劫持。

       以render是返回RouterContext为例，继续往下走流程。

       RouterContext，终于到了最后一步，完成了组件树的返回。

       ```javascript
       render() {
           const { location, routes, params, components, router } = this.props
           let element = null // 最终组件树
           if (components) {
             // 从最右边的元素开始迭代，完成创建react元素的过程，然后在下一轮作为子元素,放入props.children
             element = components.reduceRight((element, components, index) => {
               if (components == null)
                 return element // Don't create new children; use the grandchildren.
       
               const route = routes[index]
               const routeParams = getRouteParams(route, params)
               // 把route加入props，所以可以访问push等API
               const props = {
                 location,
                 params,
                 route,
                 router,
                 routeParams,
                 routes
               }
       
               if (isReactChildren(element)) { // 当前元素是react元素
                 props.children = element
               } else if (element) {
                 for (const prop in element)
                   if (Object.prototype.hasOwnProperty.call(element, prop))
                     props[prop] = element[prop]
               }
       
               if (typeof components === 'object') {
                 const elements = {}
                 /**
                  * 只遍历components本身的属性，借用hasOwnProperty更安全（obj.hasOwnProperty可能报错、或者被覆盖）
                 */
                 for (const key in components) {
                   if (Object.prototype.hasOwnProperty.call(components, key)) {
                     elements[key] = this.createElement(components[key], {
                       key, ...props
                     })
                   }
                 }
       
                 return elements
               }
       
               return this.createElement(components, props)
             }, element)
           }
       
           invariant(
             element === null || element === false || React.isValidElement(element),
             'The root route must render a single element'
           )
       
           return element
         }
       ```

       到这里整个初始化过程完成，剩下的事情交给React取渲染组件就可以了，渲染过程会在React章节做分析。

##### 2.2 路由改变

这里说的改变路由是指通过a链接、history.push、history.replace，改变历史栈中的路由。

1. a链接

   a链接改变路由触发组件更新需要监听PopStateEvent、HashChangeEvent。

   PopStateEvent

   ```javascript
   export const startListener = (listener) => {
     const handlePopState = (event) => {
       if (event.state !== undefined) // Ignore extraneous popstate events in WebKit
         listener(_createLocation(event.state))
     }
   // 注意：调用history.pushState()或者history.replaceState()不会触发popstate事件. popstate事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者在JavaScript中调用history.back()、history.forward()、history.go()方法)，此外，a 标签的锚点也会触发该事件.
     addEventListener(window, PopStateEvent, handlePopState)
   
     return () =>
       removeEventListener(window, PopStateEvent, handlePopState)
   }
   ```

   HashChangeEvent

   ```javascript
   export const startListener = (listener, queryKey) => {
       const handleHashChange = () => {
   
           if (!ensureSlash())
               return // Hash path must always begin with a /
   
               const currentLocation = getCurrentLocation(queryKey)
   
           if (prevLocation && currentLocation.key && prevLocation.key === currentLocation.key)
               return // Ignore extraneous hashchange events
   
               prevLocation = currentLocation
   
           listener(currentLocation)
       }
   ```

   最终都是执行listener，下面是看看listener是谁

   creatBrowserHistory

   ```javascript
   stopListener = BrowserProtocol.startListener(
   	history.transitionTo
   )
   ```

   createHashHistory

   ```javascript
   stopListener = HashProtocol.startListener(
       history.transitionTo,
       queryKey
   )
   ```

   都指向了transitionTo。

2. push

   ```javascript
   const push = (input) =>
   transitionTo(createLocation(input, PUSH))
   ```

   

3. replace

   ```javascript
   const replace = (input) =>
   transitionTo(createLocation(input, REPLACE))
   ```

   transitionTo的执行过程在browserHistory分析过程中已分析。

### 3. 小技巧

1. 开启监听与取消监听

   ```javascript
   // startListener函数执行后触发开启监听，startListener函数执行取消监听，这样看起来结构更紧密
   export const startListener = (listener) => {
     const handlePopState = (event) => {
       if (event.state !== undefined) // Ignore extraneous popstate events in WebKit
         listener(_createLocation(event.state))
     }
     addEventListener(window, PopStateEvent, handlePopState)
     return () =>
       removeEventListener(window, PopStateEvent, handlePopState)
   }
   ```

   ```javascript
   //  通过闭包，在函数体里添加一个listener，在函数执行后返回的函数体里把当前listener去除
   const listen = (listener) => {
       listeners.push(listener)
       return () =>
       listeners = listeners.filter(item => item !== listener)
   }
   ```

   

2. js生成随机数

   ```javascript
   /**
        * Math.random()生成一个0-1之间的随机数
        * toString(36)将随机数转为36机制然后转为字符串，36进制包括了10个数字26个英文字母
        * substr(2, keyLength)，从第二位开始，一共keyLength位
       */
   Math.random().toString(36).substr(2, keyLength)
   ```

3. replace中第二个参数可以是函数

   ```javascript
   search.replace(
       new RegExp(`([?&])${key}=[a-zA-Z0-9]+(&?)`),
       // match整个正则匹配到的search的部分，prefix正则表达式中第一个分组([?&])匹配到的search部分，suffix第二个分组(&?)匹配到的search部分
       // 返回值将替换调search中match对应的部分
       (match, prefix, suffix) => (
           prefix === '?' ? prefix : suffix
       )
   )
   ```

4. 解析遍历对象key时各个浏览器顺序不一样的问题，具体原因见[博文](https://www.cnblogs.com/everlose/p/12501222.html)

   ```javascript
   Object.keys(obj).sort()
   ```

5. 取字符串的第一个字符

   ```javascript
   /**
        * charAt比[]的兼容性更好（IE7）
        * 
       */
   location.pathname.charAt(0) !== '/'
   ```

6. 函数的参数个数

   ```javascript
   // 要执行func函数，func由另外的库定义，如果func函数想同步执行，那么就自动调用func2，否则让另外的库自己调用，这个功能可以通过func.length来实现，函数的length属性指的是形参个数，即声明时func通过参数的个数就传达了装饰者模式的装饰函数一个自己需要同步还时异步
   function func(name, sex){}
   function wrapper(func) {
       let sync = true
       if(func.length === 2) {
           sync = true
       } else {
           sync = false
       }
       return function({func2, ...rest}) {
           func.apply(this, [func2,...rest])
           if(sync) {
               func2()
           }
       }
   }
   wrapper(func)({func2: function(){ console.log('同步执行') }})
   ```

7. 遍历对象自身属性（不包括原型属性）

   ```javascript
    /**
              * 只遍历components本身的属性，借用hasOwnProperty更安全（obj.hasOwnProperty可能报错、或者被覆盖）
             */
   for (const key in components) {
       if (Object.prototype.hasOwnProperty.call(components, key)) {
       }
   }
   ```

   

### 4. 总结

作者：[ryanflorence](https://twitter.com/ryanflorence),

