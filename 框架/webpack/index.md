## 初始化

#### 命令行参数

```javascript
require("./config-yargs")(yargs);
```

webpack使用了yargs作为命令行交互的工具，config-yargs是对yargs的一部分配置，另一部分配置在yargs.options中配置，配置项中包含了命令行参数有哪些。

```javascript
yargs.parse(process.argv.slice(2), (err, argv, output) => { 
// 省略....
})
```

从命令行参数的第三个参数开始，output为yargs的输出信息。

#### 配置项对象

```javascript
var options = require("./convert-argv")(yargs, argv);
```

从配置文件和命令行参数中获取最终的配置项对象options。

#### 输出配置

```javascript
var outputOptions = options.stats;
if(typeof outputOptions === "boolean" || typeof outputOptions === "string") {
    outputOptions = statsPresetToOptions(outputOptions);
} else if(!outputOptions) {
    outputOptions = {};
}
```

配置打包时输出那些信息，stats取值none、verbose、detailed、minimal、errors-only等等。

```javascript
if(options.plugins && Array.isArray(options.plugins)) {
    compiler.apply.apply(compiler, options.plugins); // 
}
```

这里将配置文件中配置的插件注册到compiler的plugins中。

## 编译

编译过程完成入口文件的读取，并通过入口文件的内容得到语法树，通过语法树分析出入口文件的依赖项，然后依次寻找依赖项的依赖项。本篇文章的目的一方面是通过对主要事件的梳理了解整个编译过程，再就是对某些事件中的亮点进行发掘和学习、总结。

以下通过对compiler主要生命周期的梳理来理解整个过程。

### environment

environment生命周期，webpack自身在这个生命周期并没有注册回调，但是这个生命周期之前完成了编译环境的初步配置，包括从命令行解析参数、根据命令行和配置文件中的参数合并到一个配置对象中、配置对象规范性校验、实例化compiler、把配置对象中的plugins注册到对应的生命周期，下面对这一系列的步骤中的重要环节分析。

1. 配置对象的合并

   - 从命令行读取配置 比如一些组合配置项的简写，一个d命令就相当于配置了debug = true output-pathinfo = true 等等。

   - 加载配置文件,获取配置文件的对应，这里涉及没有定义配置文件的查找。

   - 根据配置项来定义对应的选项。

2. 自定义插件的注册

   由于在配置文件中已经对插件进行了实例化，这里只是执行实例对象的apply。

### after-environment

同environment，这个生命周期本身并没有钩子函数，但是这个生命周期后紧接着就是对webpack内部插件的注册，这些插件涉及了后面的很多生命周期，构成了webpack打包的骨架，具体注册那些插件取决于配置对象，比如根据entry的类型决定是单入口、多入口、动态入口。

### normalModuleFactory

这个生命周期之前完成了NormalModuleFactory的实例化，NormalModuleFactory顾名思义会生成NormalModule，即模块，而通过模块的链式查找，wepack会从入口文件根据依赖遍历所有相关文件，所以在模块创建过程中主要是在NormalModuleFactory进行的。

### thisCompilation

这个生命周期之前完成了compilation对象的创建，而compilation则包括模块创建、模块构建、文件输出等重要环节，而这个生命周期本身会注册compilation运行时需要的一系列插件。

### make

```javascript
compiler.plugin("make", (compilation, callback) => {
    const dep = SingleEntryPlugin.createDependency(this.entry, this.name);
    compilation.addEntry(this.context, dep, this.name, callback);
});
```

这里先生成一个入口文件信息对象，包括入口所在路径和名称，名称是死值main，然后调用compilation的addEntry方法，该方法的作用一方面是构建入口模块，并把入口模块添加到compilation的entries属性中，再就是链式寻找所有依赖的模块。

#### 构建入口模块

构建入口模块分为创建模块，读取模块内容，解析模块内容，添加模块依赖这四部分。

1. 创建入口模块

   这里先创建入口文件对应的依赖即SingleEntryDependency，包括了入口文件的路径信息，接着就是入口模块的创建过程。

   - 创建一个slot，这个slot在模块创建完毕再补充一些信息，而slot本身会放入_preparedEntrypoints。
   - webpack通过semaphore对象维护一个限制同时最多处理资源的数量，每运行一个资源数量减一，释放后数量加一，为0后的资源维护到等待队列中，释放资源时检查等待队列。
   - 然后是进入normalModuleFactory.create方法，在resolver生命周期会分离、解析内联的loader，把配置文件中的loader和当前资源做匹配，把匹配到loader存起来，然后是把内联loader和配置文件中匹配的loader进行合并，在调用回调函数前有一个重要的参数parser，这个参数在解析源文件为语法树时使用，然后进入factory生命周期，这里会根据之前的loader然后创建入口模块，入口模块创建后

2. 读取模块内容

   在compilation的buildModule方法里，入口模块调用了自身的buile->doBuild方法，最终会调用runLoaders，runLoaders方法前会生成loaderContext对象，这个对象包括当前资源的loaders，当前使用的是哪个loader，在webpage官网文档上有对这个对象的[介绍](https://webpack.js.org/api/loaders/#pitching-loader)，这个对象除了在调用loader时有用，在loader中也可以访问这个对象，这个对象提供了让loader和webpack通信的机会，比如告诉webpack打印一些警告、错误、获取异步API等。

   runLoaders方法完成了加载loader文件，运行pitch方法，读取文件内容，运行loader方法，pitch方法也是loader可以暴露的一个接口，这个接口的作用是让loader在读取文件前记录一些数据，然后读取文件，然后可以把记录的数据传给loader方法，另外一个作用是一旦pitch方法有返回值那么就停止了后面loader的执行，pitch的执行顺序是从左到右，loader的执行顺序是从右到左，这个过程有点像浏览器事件冒泡和捕获阶段。

   而runLoaders最终返回得是二进制的源文件内容，然后把二进制转为字符串内容，接下来是把源文件内容转为AST语法树。

   解析模块内容

   ```javascript
   parse(source, initialState) {
       let ast;
       const comments = [];
       for(let i = 0, len = POSSIBLE_AST_OPTIONS.length; i < len; i++) {
           if(!ast) {
               try {
                   comments.length = 0;
                   POSSIBLE_AST_OPTIONS[i].onComment = comments;
                   ast = acorn.parse(source, POSSIBLE_AST_OPTIONS[i]);
               } catch(e) {
                   // ignore the error
               }
           }
       }
       if(!ast) {
           // for the error
           ast = acorn.parse(source, {
               ranges: true,
               locations: true,
               ecmaVersion: ECMA_VERSION,
               sourceType: "module",
               plugins: {
                   dynamicImport: true
               },
               onComment: comments
           });
       }
       if(!ast || typeof ast !== "object")
           throw new Error("Source couldn't be parsed");
       const oldScope = this.scope;
       const oldState = this.state;
       const oldComments = this.comments;
       this.scope = {
           inTry: false,
           definitions: [],
           renames: {}
       };
       const state = this.state = initialState || {};
       this.comments = comments;
       if(this.applyPluginsBailResult("program", ast, comments) === undefined) {
           /* 为变量重命名,$variable存入this.scope.rename,旧名存入 this.scope.definitions*/
           this.prewalkStatements(ast.body);
           this.walkStatements(ast.body);
       }
       this.scope = oldScope;
       this.state = oldState;
       this.comments = oldComments;
       return state;
   }
   ```

   这部分代码是webpack能根据入口文件然后找出所有依赖的关键，首先是构建入口模块的语法树，见以下代码。

   ```javascript
    ast = acorn.parse(source, POSSIBLE_AST_OPTIONS[i]);
   ```

   ast是一个javascript对象，该对象是根据文件内容生成的语法树。

3. 添加模块依赖

   ```javascript
    this.prewalkStatements(ast.body);
    this.walkStatements(ast.body);
   ```

   这部分代码就是根据语法树寻找入口文件的依赖并添加到模块的 dependencies 属性中。

#### 添加入口模块

```javascript
this._addModuleChain(context, entry, (module) => {
    entry.module = module;
    this.entries.push(module);
    module.issuer = null;

}, (err, module) => {
    if(err) {
        return callback(err);
    }

    if(module) {
        slot.module = module;
    } else {
        const idx = this.preparedChunks.indexOf(slot);
        this.preparedChunks.splice(idx, 1);
    }
    return callback(null, module);
});
```

其中两个回调函数一个是把入口模块添加到entries中，一个是所有模块解析完毕后把入口模块添加到preparedChunks中，preparedChunks为seal事件中打包chunk做准备。

#### 链式寻找所有依赖

```javascript
processModuleDependencies(module, callback) {
		const dependencies = [];

		function addDependency(dep) {
			for(let i = 0; i < dependencies.length; i++) {
				if(dep.isEqualResource(dependencies[i][0])) {
					return dependencies[i].push(dep);
				}
			}
			dependencies.push([dep]);
		}

		function addDependenciesBlock(block) {
			if(block.dependencies) {
				iterationOfArrayCallback(block.dependencies, addDependency);
			}
			if(block.blocks) {
				iterationOfArrayCallback(block.blocks, addDependenciesBlock);
			}
			if(block.variables) {
				iterationBlockVariable(block.variables, addDependency);
			}
		}
		addDependenciesBlock(module);
		this.addModuleDependencies(module, dependencies, this.bail, null, true, callback);
	}
```

遍历入口文件的依赖。addModuleDependencies方法代码和_addModuleChain基本类似，最终遍历所有依赖模块，存入__modules中。

## Hooks

### Hooks的重要性

众所周知wepack是由一系列的插件来组成的，包括webpack内部插件和外部插件，而插件的注册和执行的核心就是依赖"tapable"即Hooks，所有知道了Hooks的执行原理无论是写插件、还是读插件都有由事半功倍的效果。

### Hooks的分类

1. 按执行的逻辑分类

   basic，基本的Hooks，如果Hooks的名字里没有bail、waterfall、loop，哪这个Hooks就是基本的Hooks，Hooks的回调函数是一个接着一个执行。

   waterfall，同上，瀑布流式的一个执行完了再紧接着执行下一个，函数间可以传递参数。

   bail，可提前退出的Hooks，如果某个Hooks中的回调函数返回了非undefined的值，那么就可以提前推出了。

   loop，可循环执行的的Hooks，如果某个Hooks的回调函数返回了非undefined的值，那么从第一个回调重新执行，直到所有回调都返回undefined。

2. 按注册的类型分类

   sync，这类Hooks只能通过tap注册插件，执行时可以用call、callAsync、promise，但是执行时都是一个接着一个执行，但是执行callAsync时必须传递个回调函数作为第二个参数

   ```
   Hook.callAsync({}, () => {})
   ```

   asyncSeries，可以通过同步的(tap)、基于回调的(tapAsync)、promise(tapPromise)的函数注册，但是执行时只能用callAsync或者promise，对于通过tapAsync注册进回调，会传递一个执行下一个异步函数的回调，对于通过tap注册的回调则没有这个参数回调。

   ```javascript
   const Hook = new AsyncSeriesHook(["compiler"])
     Hook.tapAsync('foo', (a, next) => {
       console.log('1', next)
       return 'a'
     })
     Hook.tap('foo1', (b) => {
       console.log('2', b)
       return 'b'
     })
     Hook.tapAsync('foo2', (c) => {
       console.log('3', c)
       return 'c'
     })
     Hook.callAsync()
   // 最后生成的执行函数
   (function anonymous(compiler, _callback
   ) {
     "use strict";
     var _context;
     var _x = this._x;
     function _next0() {
       var _fn1 = _x[1];
       var _hasError1 = false;
       try {
         _fn1(compiler); // 普通执行
       } catch (_err) {
         _hasError1 = true;
         _callback(_err);
       }
       if (!_hasError1) {
         var _fn2 = _x[2];
         _fn2(compiler, _err2 => {
           if (_err2) {
             _callback(_err2);
           } else {
             _callback();
           }
         });
       }
     }
     var _fn0 = _x[0];
     _fn0(compiler, _err0 => { // 带回调执行
       if (_err0) {
         _callback(_err0);
       } else {
         _next0();
       }
     });
   
   })
   ```

   asyncParallel，可以通过同步的(tap)、基于回调的(tapAsync)、promise(tapPromise)的函数注册，但是调用异步函数是同时调用，理解了同步和异步的区别，那么异步的“series”和“parallel”就好区分了，“series”需要在第一个回调里执行第二个参数函数来调起下一个异步函数的函数，而“parallel”所有的异步函数同时执行，而这里回调函数第二个参数函数的作用仅是为了统计调用完所有异步函数后调用最后的回调，有点绕，必须上代码。

   ```javascript
   const Hook = new AsyncParallelHook(["compiler"])
     Hook.tapAsync('foo', (a, next) => {
       console.log('1', next)
       return 'a'
     })
     Hook.tap('foo1', (b) => {
       console.log('2', b)
       return 'b'
     })
     Hook.tapAsync('foo2', (c) => {
       console.log('3', c)
       return 'c'
     })
     Hook.callAsync()
   // 生成执行代码如下
   (function anonymous(compiler, _callback
   ) {
     "use strict";
     var _context;
     var _x = this._x;
     do {
       var _counter = 3;
       var _done = () => {
         _callback();
       };
       if (_counter <= 0) break;
       var _fn0 = _x[0];
       _fn0(compiler, _err0 => {
         if (_err0) {
           if (_counter > 0) {
             _callback(_err0);
             _counter = 0;
           }
         } else {
           if (--_counter === 0) _done();
         }
       });
       if (_counter <= 0) break;
       var _fn1 = _x[1];
       var _hasError1 = false;
       try {
         _fn1(compiler);
       } catch (_err) {
         _hasError1 = true;
         if (_counter > 0) {
           _callback(_err);
           _counter = 0;
         }
       }
       if (!_hasError1) {
         if (--_counter === 0) _done();
       }
       if (_counter <= 0) break;
       var _fn2 = _x[2];
       _fn2(compiler, _err2 => {
         if (_err2) {
           if (_counter > 0) {
             _callback(_err2);
             _counter = 0;
           }
         } else {
           if (--_counter === 0) _done();
         }
       });
     } while (false);
   
   })
   ```

   

### intercept

intercept截断器,在tap的某些阶段执行前执行

- register方法在添加截断器时可以获取每个taps并修改

- call 当Hooks被触发时调用触发,可以获取Hooks参数

- tap 当Hooks添加插件时触发此时不能修改taps

- loop 当loop Hooks执行时调用

### Context

Hooks和intercept可以选择性的访问这个值，这个值会被传给后续插件和截断器。

## Loaders

#### 关于什么是loaders

>  Out of the box, webpack only understands JavaScript and JSON files. **Loaders** allow webpack to process other types of files and convert them into valid [modules](https://webpack.js.org/concepts/modules) that can be consumed by your application and added to the dependency graph. 

也就是有了loaders，webpack可以打包除了js和json外其它类型的文件，可以把loaders看成是weppack的补充，让它能所不能。

#### 基本用法

##### 使用方式

1. 配置方式

   ```javascript
   const path = require('path');
   
   module.exports = {
     output: {
       filename: 'my-first-webpack.bundle.js'
     },
     module: {
       rules: [
         { test: /\.txt$/, use: 'raw-loader' }
       ]
     }
   };
   ```

   test属性告诉webpack匹配哪种文件，test可以使用正则去匹配文件，如果是带引号的正则匹配的是一个文件，不带引号的是匹配所有匹配到的文件；use属性告诉webpack使用哪种loader，use可以用字符串匹配一个loader，也可以用数组匹配多个loader。rules内配置的loader的执行顺序是从下到上，从右到左。

2. 内联方式

   ```javascript
   import styles from 'style-loader!css-loader?modules!./index.css'
   ```

   !代表优先级，比如!代表比普通的配置的loader优先级高，!!比所有的loader、preLoader、postLoader的优先级高，-!代表比除了postLoader的loader、preLoader的优先级高；?后可以加参数代表选项。

3. CLI方式

   ```javascript
   webpack --module-bind pug-loader --module-bind 'css=style-loader!css-loader'
   ```

##### loader特性

1. **多个loader可以是链式的，每个loader多能对源码转换，并且是逆序执行的。**
2. **loader可以异步也可以同步。**
3. **loader运行在node.js环境，可以做任何node允许的事。**
4. loader可以有个option属性，代表loader的配置。
5. 正常的模块除了可以导出模块外还可以通过package.json中的loader属性导出loader。
6. loader可以创建额外的文件。

##### css-loader

css-loader把css代码转换成javascript可以使用的代码。

webpack.config.js

```javascript
const path = require('path');

module.exports = {
    entry: './src/index.js',
    output: {
        filename: '[name].js',
        path: path.resolve(__dirname, './dist'),
    },
    mode: 'production',
    module: {
        rules: [
            {
                test: /\.css$/i,
                loader: 'css-loader'
            },
        ],
    }
}
```

index.css

```javascript
.initstyle {
	background: url(http://localhost/1.jpg);
	background-size: 100% 100%;
}
```

index.js

```javascript
import styles from './index.css'
console.log('say hi', styles.toString())
/*
say hi .initstyle {
	background: url(http://localhost/1.jpg);
	background-size: 100% 100%;
}
*/
```

#### 机制

1. 获取loader

   - 配置文件中的loader（配置loader）的获取

     ```javascript
     this.ruleSet = new RuleSet(options.defaultRules.concat(options.rules));
     ```

     ![1594665133584](C:\Users\jiang\AppData\Roaming\Typora\typora-user-images\1594665133584.png)

     其中的defaultRules。

     ruleSet对象存储了所有配置loader的信息，并对配置loader做了校验，把资源路径检测封装到resource属性。

     ```javascript
     const checkResourceSource = newSource => {
         if (resourceSource && resourceSource !== newSource) {
             throw new Error(
                 RuleSet.buildErrorMessage(
                     rule,
                     new Error(
                         "Rule can only have one resource source (provided " +
                         newSource +
                         " and " +
                         resourceSource +
                         ")"
                     )
                 )
             );
         }
         resourceSource = newSource;
     };
     ```

     这里是一个资源来源的唯一性校验，如果用了test、include、exclude做了资源来源的限定，那么就不能用resource做限定了，具体逻辑是如果出现了test、include、exclude就调用checkResourceSource，并为resourceSource赋值，如果碰到了resource，再次调用checkResourceSource就会抛出error。

     ```javascript
     const keys = Object.keys(rule).filter(key => {
         return ![
             "resource",
             "resourceQuery",
             "compiler",
             "test",
             "include",
             "exclude",
             "issuer",
             "loader",
             "options",
             "query",
             "loaders",
             "use",
             "rules",
             "oneOf"
         ].includes(key);
     });
     for (const key of keys) {
         newRule[key] = rule[key];
     }
     ```

     这里过滤掉了范围外的key。

     ```javascript
     newRule.resource = RuleSet.normalizeCondition(condition)
     /**************上方代码展示封装的赋值，下方展示封装过程*******************************/
     static normalizeCondition(condition) {
         if (!condition) throw new Error("Expected condition but got falsy value");
         if (typeof condition === "string") {
             return str => str.indexOf(condition) === 0;
         }
         if (typeof condition === "function") {
             return condition;
         }
         if (condition instanceof RegExp) {
             return condition.test.bind(condition);
         }
         if (Array.isArray(condition)) {
             const items = condition.map(c => RuleSet.normalizeCondition(c));
             return orMatcher(items);
         }
         if (typeof condition !== "object") {
             throw Error(
                 "Unexcepted " +
                 typeof condition +
                 " when condition was expected (" +
                 condition +
                 ")"
             );
         }
     
         const matchers = [];
         Object.keys(condition).forEach(key => {
             const value = condition[key];
             switch (key) {
                 case "or":
                 case "include":
                 case "test":
                     /* 存入一个函数,该函数传入路径返回该路径中是否包含这个值得布尔 */
                     if (value) matchers.push(RuleSet.normalizeCondition(value));
                     break;
                 case "and":
                     if (value) {
                         const items = value.map(c => RuleSet.normalizeCondition(c));
                         matchers.push(andMatcher(items));
                     }
                     break;
                 case "not":
                 case "exclude":/* 存入一个函数,该函数传入路径返回该路径中是否不包含这个值得布尔 */
                     if (value) {
                         const matcher = RuleSet.normalizeCondition(value);
                         matchers.push(notMatcher(matcher));
                     }
                     break;
                 default:
                     throw new Error("Unexcepted property " + key + " in condition");
             }
         });
         if (matchers.length === 0) {
             throw new Error("Excepted condition but got " + condition);
         }
         if (matchers.length === 1) {
             return matchers[0];
         }
         return andMatcher(matchers);
     }
     ```

     首先是对condition类型的判断，如果是字符串检测是否包含，函数直接返回，正则绑定检测函数，数组的话就循环调用。再就是根据关键词组装检测逻辑：

     - 如果是“or”、“include”、“test”，则把匹配函数（如“str => str.indexOf(condition) === 0”）存入matchers数组。
     - 如果是“and”，则把and的值依次遍历汇总到items，最后由andMatcher连接（下面解释）。
     - 如果是“not”、“exclude”，则把不匹配函数存入matcher数组。
     - 最后是andMatcher，闭包返回一个函数,该函数匹配items的所有项,如果有你匹配的则返回false,全部匹配返回true。

   - 内联的loader的获取，[方式](https://webpack.js.org/concepts/loaders/#inline)，另外有一个issuer的配置，是用配置某个文件的所有的请求资源应用的loader的，[见](https://stackoverflow.com/questions/46762439/rule-issuer-in-webpack-what-properties-belong-to-it)

     ```javascript
     this.hooks.resolver.tap("NormalModuleFactory", () => (data, callback) => {
         const contextInfo = data.contextInfo;
         const context = data.context;
         const request = data.request;
     
         const loaderResolver = this.getResolver("loader");
         const normalResolver = this.getResolver("normal", data.resolveOptions);
     
         let matchResource = undefined;
         let requestWithoutMatchResource = request;
         /* 如果我们配置的是js文件但是想用下css的loader那么在请求路径前加1.css!=!就醒了						https://webpack.docschina.org/api/loaders/#inline-matchresource */
         /* 匹配第一个不是!后面是!=!的字符串,例如"^./test.css!=!hello.js" */
         const matchResourceMatch = MATCH_RESOURCE_REGEX.exec(request);
         if (matchResourceMatch) {
             /* 非!的匹配对应[^!]+ */
             matchResource = matchResourceMatch[1]; // "^./test" 
             /* 匹配./ 或者 ../ */
             if (/^\.\.?\//.test(matchResource)) { // 包含路径,最终得到相对路径
                 matchResource = path.join(context, matchResource);
             }
             /* !=!后面的部分"hello" */
             requestWithoutMatchResource = request.substr(
                 matchResourceMatch[0].length
             );
         }
         // 因为后面要进行字符串替换,所以提前判断可以有那些的loaders
         const noPreAutoLoaders = requestWithoutMatchResource.startsWith("-!");
         const noAutoLoaders =
               noPreAutoLoaders || requestWithoutMatchResource.startsWith("!");
         const noPrePostAutoLoaders = requestWithoutMatchResource.startsWith("!!");
     
         let elements = requestWithoutMatchResource
         .replace(/^-?!+/, "") // 去掉开头的!或者-!或者--!或者-!!等等
         .replace(/!!+/g, "!") // 把路径中所有的!!换成!
         .split("!");
         let resource = elements.pop(); // 资源路径
         elements = elements.map(identToLoaderRequest); // 把每个loader的路径和options分隔到一个对象里
         debugger
         asyncLib.parallel(
             [
                 callback =>
                 this.resolveRequestArray(
                     contextInfo,
                     context,
                     elements,
                     loaderResolver,
                     callback
                 ),
                 callback => {
                     if (resource === "" || resource[0] === "?") {
                         return callback(null, {
                             resource
                         });
                     }
                     /* 将资源路径解析为绝对路径 */
                     normalResolver.resolve(
                         contextInfo,
                         context,
                         resource,
                         {},
                         (err, resource, resourceResolveData) => {
                             if (err) return callback(err);
                             callback(null, {
                                 resourceResolveData,
                                 resource
                             });
                         }
                     );
                 }
             ],
             (err, results) => { /* 省略代码 */ })
     ```

     这里分为三部分（其中一部分涉及MATCH_RESOURCE_REGEX不常用这里暂时不提）

     ```javascript
     const noPreAutoLoaders = requestWithoutMatchResource.startsWith("-!");
     const noAutoLoaders =
           noPreAutoLoaders || requestWithoutMatchResource.startsWith("!");
     const noPrePostAutoLoaders = requestWithoutMatchResource.startsWith("!!");
     ```

     ![1594665067347](C:\Users\jiang\AppData\Roaming\Typora\typora-user-images\1594665067347.png)

     这里涉及内联的loader对配置文件中的三种loader的禁用关系，由于preloader和postloader已弃用所以这里不分析。

     ```javascript
     let elements = requestWithoutMatchResource
         .replace(/^-?!+/, "") // 去掉开头的!或者-!或者--!或者-!!等等
         .replace(/!!+/g, "!") // 把路径中所有的!!换成!
         .split("!");
         let resource = elements.pop(); // 资源路径
         elements = elements.map(identToLoaderRequest); // 把每个loader的路径和options分隔到一个对象里
         debugger
         asyncLib.parallel(
             [
                 callback =>
                 this.resolveRequestArray(
                     contextInfo,
                     context,
                     elements,
                     loaderResolver,
                     callback
                 ),
                 callback => {
                     if (resource === "" || resource[0] === "?") {
                         return callback(null, {
                             resource
                         });
                     }
                     /* 将资源路径解析为绝对路径 */
                     normalResolver.resolve(
                         contextInfo,
                         context,
                         resource,
                         {},
                         (err, resource, resourceResolveData) => {
                             if (err) return callback(err);
                             callback(null, {
                                 resourceResolveData,
                                 resource
                             });
                         }
                     );
                 }
             ],
             (err, results) => { /* 省略代码 */ })
     ```

     这部分代码完成两个功能从资源路径匹配出loader，将loader解析为绝对路径。

   - 两者的合并

     ```javascript
     if (matchResource === undefined) {
         loaders = results[0].concat(loaders, results[1], results[2]);
     } else {
         loaders = results[0].concat(results[1], loaders, results[2]);
     }
     ```

     这里完成了合并，合并的方式很简单，也就是如果我们如果内联和配置文件中都声明了某个loader，那么该loader会执行两遍。

2. loader的预处理-执行pitch阶段

   什么是loader的pitch阶段？pitch是loaderexports的一个方法，该方法在loader处理内容前被调用，官网有个很形象的流程。

   ```javascript
   module.exports = {
     //...
     module: {
       rules: [
         {
           //...
           use: [
             'a-loader',
             'b-loader',
             'c-loader'
           ]
         }
       ]
     }
   };
   ```

   可能有这样的调用过程

   ```diff
   |- a-loader `pitch`
     |- b-loader `pitch`
       |- c-loader `pitch`
         |- requested module is picked up as a dependency
       |- c-loader normal execution
     |- b-loader normal execution
   |- a-loader normal execution
   ```

   其中的pitch就是pitch阶段，normal就是正式处理模块内容，中间的是读取源文件，可以看到pitch和normal有相反的调用顺序。

   为什么要有pitch？pitch可以在loader正式处理内容前被调用，pitch阶段的data参数会传递给normal阶段。

   ```javascript
   function iteratePitchingLoaders(options, loaderContext, callback) {
       // abort after last loader
       if(loaderContext.loaderIndex >= loaderContext.loaders.length)
           return processResource(options, loaderContext, callback);
   
       var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];
   
       // iterate
       if(currentLoaderObject.pitchExecuted) {
           loaderContext.loaderIndex++;
           return iteratePitchingLoaders(options, loaderContext, callback);
       }
   
       // load loader module
       loadLoader(currentLoaderObject, function(err) {
           if(err) {
               loaderContext.cacheable(false);
               return callback(err);
           }
           var fn = currentLoaderObject.pitch;
           currentLoaderObject.pitchExecuted = true;
           if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);
   
           runSyncOrAsync(
               fn,
               loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
               function(err) {
                   if(err) return callback(err);
                   var args = Array.prototype.slice.call(arguments, 1);
                   if(args.length > 0) {
                       loaderContext.loaderIndex--;
                       iterateNormalLoaders(options, loaderContext, args, callback);
                   } else {
                       iteratePitchingLoaders(options, loaderContext, callback);
                   }
               }
           );
       });
   }
   ```

   下面对这部分核心代码展开剖析。

   ```javascript
   var fn = currentLoaderObject.pitch;
   currentLoaderObject.pitchExecuted = true;
   if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);
   runSyncOrAsync(
       fn,
       loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
       function(err) {
           if(err) return callback(err);
           var args = Array.prototype.slice.call(arguments, 1);
           if(args.length > 0) {
               loaderContext.loaderIndex--;
               iterateNormalLoaders(options, loaderContext, args, callback);
           } else {
               iteratePitchingLoaders(options, loaderContext, callback);
           }
       }
   );
   ```

   首先去加载当前的loader，并尝试着执行pitch函数，然后有以下结果：

   1. 当前loader没有pitch函数或者pitch函数没有返回值，那么继续加载下个loader并尝试执行pitch函数。
   2. 当前loader的pitch函数有返回值，那么就终止加载下个loader，开始执行iterateNormalLoaders方法。
   3. 另外，如果所有的loader都被加载完毕，那么就执行processResource，请求资源文件内容，然后执行iterateNormalLoaders方法。

   

   3. 执行loader-执行normal阶段

   ```javascript
   if(loaderContext.loaderIndex < 0)
       return callback(null, args);
   
   var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];
   
   // iterate
   if(currentLoaderObject.normalExecuted) {
       loaderContext.loaderIndex--;
       return iterateNormalLoaders(options, loaderContext, args, callback);
   }
   
   var fn = currentLoaderObject.normal;
   currentLoaderObject.normalExecuted = true;
   if(!fn) {
       return iterateNormalLoaders(options, loaderContext, args, callback);
   }
   
   convertArgs(args, currentLoaderObject.raw);
   
   runSyncOrAsync(fn, loaderContext, args, function(err) {
       if(err) return callback(err);
   
       var args = Array.prototype.slice.call(arguments, 1);
       iterateNormalLoaders(options, loaderContext, args, callback);
   });
   ```

   调用iterateNormalLoaders方法有两种情况：

   1. iterateNormalLoaders是在iteratePitchingLoaders函数内被调用的，那么此时没有读取真正的资源文件内容，模块资源属性是最后一个执行pitch方法的loader的返回值，然后用该值作为模块资源属性传递给已加载的loader（除了最后一个执行pitch方法的loader）。
   2. iterateNormalLoaders是在processResource函数内调用的，那么此时读取了真正的资源文件内容，所有的loader都已被加载并尝试了执行pitch函数，此时会从最后一个loader起调用所有loader的module方法，来完成资源的转化。



### 输出

webpack支持CommonJs和ES Module。

CommonJs：这里webpack没有特殊处理，因为node是本来支持CommonJs。

ES Module：

1. 定义__esModule的value为true，告诉第三方库这是ES module。
2. 为export定义getter，这里是一个闭包，所以在模块内模块外都可以访问并修改这个export值。

### source-map模式

webpack为了源代码的执行在bundle里注入了许多代码，这些代码大部分都定义到webpack_require对象上。依赖模块在webpack的构建模块已经加载了，此时只是把解析后的模块函数注入bundle。

1. eval-*，把模块分组，分组后的源文件生成字符串通过eval执行，优点是控制台可以看到文件名称以及路径（通过eval的sourceURL定义），但是看到的不是源文件，而是源文件和webpack注入的代码一起的内容，缺点是代码行不准确，开发环境推荐使用，因为再次构建时相对速度快。
2. inline-*，把分组后模块直接放入bundle，优点是可以看到源文件内容、代码行、文件名称以及路径（通过@sourceMapUrl），缺点是bundle体积大因为sourceMapUrl存储着文件路径，代码位置，而且转换这个文件或者base64比较慢所以对构建过程影响大。
3. source-map即*，优点是source-map文件分离到单独的文件，bundle包引用这个文件，所以对体积相对小，缺点是构建速度慢，原因同inline。
4. hidden-*，同source-map但是bundle包里没有引用分离的source-map文件，做错误上报使用。

### 优化

开发环境，提高构建速度

1. 开发环境用source-map用eval，构建速度快，能看到源码。
2. loader系列，
   1. loader通过include或者exclude限定查找文件范围，提高编译效率。
   2. thread-loader，把比较耗时的loader单独拎一个node 进程。
3. plugin系列：
   1. dlls，把第三方库打包到单独的文件中，不用每次打包都重复打包没有改动的第三方库。
4. optimization，通过optimization.runtimeChunk把webpack运行代码单独拎一个包。

生产环境，减小包的体积

1. 生产环境慎用source-map。
2. 开启gzip压缩。
3. CommonChunkPlugin，提取公共模块，缓存后减少二次请求时间。
4. 按需引入，使用import()函数