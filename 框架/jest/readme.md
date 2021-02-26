### 匹配器

1. 常用的匹配有，toBe相等于Object.is、toEqual相当于遍历对象属性值、toNull、toDefined、toTruthy，toFalsy。
2. 数字匹配有，toBeGreaterThan、toBeGreaterOrEqualThan、toBeLessThan、toBeLessOrEqualThan。
3. 字符串，toMatch匹配正则。
4. 抛异常，toThrow

### 异步的处理

1. 基于回调的异步

   通过test的回调函数中声明一个done参数，test知道这是要处理异步了，所以会等到done执行了才处理下一个test。

2. 基于promise

   如果期望是resolve的promise，那么在then回调里写expect，并返回promise，告诉jest这是个promise测试。

   如果期望是reject的promise，那么在catch回调里写expect，除了要返回promise还得在test中加断言“expect.assert(1)”，保证正确执行测试。

   除了用then、catch还可以用resolves、rejects后加expect来去直接执行测试。

3. await、async

   首先要在test的回调函数前加async告诉jest这是一个async函数，然后await后执行expect。

   另外也可以结合resolves、rejects使用。

### 测试前和测试后的hooks

beforeEach、afterEach在每一个test执行前都执行，因为可以通过describe定义局部作用域，在describe上级作用域定义的beforeEach、afterEach除了自己作用域内的每一个test执行前执行，describe子级作用域的test执行前也会执行上级作用域的beforeEach、afterEach。

beforeAll，afterAll和Each相对，所有的test执行完毕后才执行afterAll，所有的test执行前执行beforeAll，这里的test也是区分作用域的。

describe和test的执行顺序：所有describe的回调先执行，这个过程收集test，当所有describe执行完毕后执行test。

### mock

jest.fn能生成一个函数，该函数被调用时jest会记录下各种属性，比如参数个数、调用次数、返回值，而且还能直接模拟函数的某些方法的实现、返回值、函数名称，这个函数同样适用于模块。

### snapshot

对一段代码测试是否改变。

