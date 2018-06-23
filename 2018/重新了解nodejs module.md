# 重新了解nodejs module
> 从webpack配置文件到nodejs服务器不停机更新
## 前言
webpack流行，形形色色的loader和plugin功不可没。根据文档，loader和plugin需要安装到项目目录下，由webpack配置文件调用。然而我们的项目多是活动专题，loader，plugin和其他依赖基本一样，每次新建项目都要重新安装一遍npm模块毫无必要。为了使npm模块能被多个项目共用，需要从nodejs的module入手，进一步了解其模块机制。此外，还可以利用module机制，实现线上不停机更新。
## webpack项目的配置
一个webpack项目通常会有四种依赖模块： 
1. 文件loader。webpack会把配置文件中的loader都隐式require进来。
2. webpack插件，如热更新插件，开发服务器等等。这些插件需要在webpack配置中显式引入，与nodejs类似。
3. 项目代码中依赖的模块。这部分的引用机制由webpack实现。
4. 用于启动webpack项目的模块，如webpack，webpack-cli，webpack-dev-server等等。

 ![典型的webpack配置文件](https://nos.netease.com/knowledge/d255812d-1844-43bc-b138-f62b6fcbbd26?imageView) 

所以总结起来只有两大类依赖：a. 在命令行或者npm命令中启动webpack项目所需的依赖，b. 项目中需要的依赖。对于a类依赖，只需要把模块都安装到全局即可。对于b类依赖，需要找到一个途径使不同项目都能公用同一个npm模块。

## nodejs中的module
webpack本身运行在nodejs环境，所以问题的核心在于nodejs的模块加载机制。
当调用require引入模块时，require.resolve()方法会根据以下情况找出确切的模块位置。

>从Y路径中加载模块X：
>1. X为核心模块，返回核心模块
>2. X以“/”或“./”，“../”开头，从根目录或者目录Y+X，根据不同情况以目录或文件的形式加载x。
>3. 从node_module加载X模块 

前面两种情况都很好理解，nodejs会从指定目录加载模块，支持js，json，node文件，也支持有index或package.json文件夹。 

当需要从node_module加载时，nodejs会按照下面步骤查找出模块。
>1. 分割路径Y+X，得到从Y+X开始的各级路径列表
>2. 从路径列表中寻找是否存在node_module
>3. 假如存在node_module，尝试从中加载模块X 

此外，nodejs还会尝试从全局目录加载模块。比如NODE_PATH环境变量中指定的路径，$HOME/.node_modules，$HOME/.node_libraries以及$PREFIX/lib/node。不过文档更推荐从本地node_module目录加载，这样更快也更可靠。

所以，**为了使多个项目共用同样的loader和plugin**，我们可以把webpack项目都放到同一个工作目录下，然后在该工作目录安装npm模块。根据node_module加载机制，各项目最终都可以找到这一层的node_module，加载公用的模块。省去重复安装loader和plugin的麻烦。

## 循环引用
循环引用是一个很容易出现的问题。nodejs利用cache机制解决了循环引用的问题。当出现循环引用时，模块会没有完全执行完毕就返回了。
如官网上的文档所述，main.js引用a.js，a.js引用b.js，而在b.js中又尝试引用a.js。由于cache机制，a.js在main.js已经引用过了，所以b.js中`require('a.js')`不会导致循环执行a.js，而且会得到一份未完成的a.js导出对象。这样就解决了循环引用的问题。
附上官网的例子:
```javascript
// a.js:
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done = %j', b.done);
exports.done = true;
console.log('a done');
```
```javascript
// b.js:
console.log('b starting');
exports.done = false;
const a = require('./a.js');
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');
```
```javascript
// main.js
console.log('main starting');
const a = require('./a.js');
const b = require('./b.js');
console.log('in main, a.done=%j, b.done=%j', a.done, b.done);
```
运行main.js后得到如下结果：
```shell
$ node main.js
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done=true, b.done=true
```
## 不停机热更新
利用nodejs的动态语言特性，我们可以在运行中替换变量或者函数。结合nodejs模块管理器，可以实现一个简单的不停机热更新机制。
从文档可以知道，nodejs中的module有缓存。每次require一个模块都会先检查module.cache中是否存在缓存，假如有则不会重新运行模块代码，而是直接取用cache中的内容。所以要实现模块级的热更新，首先需要把项目按模块分成多个文件，分离成核心代码和业务代码，让业务代码支持热更新。每次程序运行到具体业务时，会去require业务代码，这样就可以在模块上面做手脚，实现不停机更新业务代码。
这方面的资料可以参照[fex的一篇文章](http://fex.baidu.com/blog/2015/05/nodejs-hot-swapping/)

关键点主要是
>* 如何更新模块代码
>* 如何使用新模块处理请求
>* 如何释放老模块的资源

更新模块只需要把cache中的旧对象删除即可，由于module.parent.children会保留模块的引用，所以还需要把module.parent.children中相应的引用去除。此外还得更加实际情况，去掉其他地方的引用，防止内存泄漏。

在此贴出简单的cleanCache函数
```javascript
function cleanCache(modulePath) {
    var module = require.cache[modulePath];
    // remove reference in module.parent
    if (module.parent) {
        module.parent.children.splice(module.parent.children.indexOf(module), 1);
    }
    require.cache[modulePath] = null;
}
```
调用时需要`cleanCache(require.resolve('./code.js'));`来清除module的缓存。

不过即使这样，还是很难100%避免闭包引起的老模块的资源无法释放的问题，所以实际线上使用时，还是需要在合适时机完全重启，避免内存泄漏或者其他问题。

## 结尾
利用nodejs的module机制，我们可以解决不同环境中模块依赖缺失导致的问题。得益于module引入的缓存机制，nodejs解决了循环引用的问题，也提高了模块加载的速度。为了实现不停机更新，我们可以手动清除旧模块的资源，强制nodejs重新加载模块，实现热更新。
