# nodejs文件操作中的一些优化

## 介绍 
半年前组里决定对线上官网完成https升级，将页面里的api接口，cdn地址，超链接都强制写成https协议。需要替换的页面很多，所以自己写了一个http2https的脚本，用于替换源码中需要https升级的各种url。因为仅仅自己线下使用，所以未考虑很多问题，其中脚本的核心功能：获取源码文件列表并替换url有着很明显的性能缺陷。
1. 获取文件列表用fs模块的同步api递归实现，阻塞event loop，处理大量文件时长时间无响应。
2. 替换url时直接将文件内容读入内存并解码，内存占用极大

考虑到nodejs在发布工具，自动化脚本中的应用，获取文件列表并替换文本是一个很常用的需求，所以针对该脚本进行优化是很有意义的。此外，也可以借此加深对generator，promise和stream的理解。

## 获取文件列表的最简单版本
遍历文件树最简单的解法莫过于递归实现，利用fs模块提供的同步api，可以轻易完成。当时https替换脚本就是这样实现的。

```javascript
let pathlist = []; // 用于存放文件列表的array
function recursive(path2Inspect) {
  let tmpls = fs.readdirSync(path2Inspect),
    dirls = [];
  tmpls = tmpls.filter((v) => !ignorels.test(v));
  tmpls.forEach((v) => {
    let absPath = path.resolve(path2Inspect, v);
    if (fs.lstatSync(absPath).isDirectory()) {
      dirls.push(absPath);
    }
    if (fs.lstatSync(absPath).isFile()) {
      pathlist.push(absPath);
    }
  });
  dirls.forEach(v => recursive(v));
}
```

- 优点：简单易懂
- 缺点：会卡死，会爆内存，无法利用nodejs的异步io

## 获取文件列表的promise递归版本
fs模块提供了文件操作的异步api，理论上可以用它们实现一个高性能的异步文件列表获取函数。获取某个路径下的文件列表需要进行几个步骤：
1. 判断该路径是文件、目录还是其他 --- 异步api
2. 获取目录路径下的路径列表 --- 异步api
3. 不断递归 

理论上步骤很清晰，实际上好难，本来异步回调函数就是个地狱了，还要加上递归......总算知道网上的解法为什么多是同步操作了。所以为了实现功能，必须先解决异步api的递归写法。

### 首先从简单的原型入手
平时用到最多的异步操作而且调用自身的功能应该就是倒计时器了。利用setTimeout和闭包，可以很快地写出一个倒计时器。
```javascript
function downCounter(timer, callback) {
	setTimeout(function ii(){
		if(timer--) {
			setTimeout(ii, 1000);
		}
		else{
			callback&&callback();
		}
	}, 1000);	
}
// 调用
downCounter(10, function(){
	console.log('end');
})
``` 

### 改造成promise的版本
为了躲开回调函数层层嵌套，而且语义不明的问题，可以改写成promise版本：
```javascript
function sleep(time) {
	return new Promise(function(resolve, reject){
		setTimeout(function(){resolve();}, time);
	});
}
function downCounter(timer) {
	if(timer--) {
		return sleep(1000).then(()=>downCounter(timer));
	}
	else {
		return Promise.resolve(); // 该函数需要返回一个promise
	}
}
// 调用 
downCounter(10).then(function(){
	console.log('end');
});
``` 

## 根据原型改造文件列表获取函数
有了倒计时器的经验，把文件列表获取函数改造成promise版本就会相对简单一些了。上述原型的关键点在于根据不同条件，return不同的Promise，并**在then里面，调用自身递归，利用闭包处理上一层传进来的变量。**
所以仿照它，可以得到我们想要的版本：
```javascript
const fs = require('fs');
const util = require('util');
const path = require('path');

const promisify_lstat = util.promisify(fs.lstat);
const promisify_readdir = util.promisify(fs.readdir);

// 递归获取目录下所有文件
// rootPath： <string> path
// @Promise: string[]  the absolute path list of files
function recurseGetFiles(rootPath) {
	let fileStore = [];
	function iterateV3(rootPath) {
		return promisify_lstat(rootPath).then(stats=>{
			if(stats.isFile()) {
				fileStore.push(rootPath);
				return;
			}
			else if(stats.isDirectory()) {
				return promisify_readdir(rootPath)
				.then(ls=>Promise.all(ls.map(p=>iterateV3(path.resolve(rootPath, p))))); 
			}
			else{
				return;
			}
		});
	}

	return iterateV3(rootPath).then(function(){
		return fileStore;
	});
};

```

利用nodejs自带的process.hrtime，发现用该函数遍历某个node_modules目录的时间大概是0.9s，而之前的同步版本需要1.6s，速度提升很多。
不过该版本也有个隐藏问题：代码中用promise.all 和map，把同一目录下的子一级路径检查函数都同时压入event loop中，如果该目录的下一级子路径过多，有可能导致内存占用很高或者poll queue过长导致卡死。所以这里也可以用array.prototype.reduce把这些promise串起来，在前面的io事件处理完成后再检查下一个路径。
```javascript
//伪代码
pathList
.reduce((lastTask, currentPath)=>lastTask.then(()=>iterateV4(currentPath)), 
	Promise.resolve()); // 最后传入resolve是为了启动整个promise链
```
这样子内存消耗情况就会下降一些了，由于是一个个路径串行处理，速度变慢不少。

## 用generator改写
promise版本的文件列表获取函数需要很大脑洞，如果环境运行使用generator，利用co模块，也可以写出比较简单的异步文件列表获取版本：
```javascript
const co = require('co');
const fs = require('fs');
const util = require('util');
const path = require('path');

const promisify_lstat = util.promisify(fs.lstat);
const promisify_readdir = util.promisify(fs.readdir);

const getFileList = co.wrap(function* (rootpath) {
	let fileStore = [];
	let pathList = [rootpath];
	let path2Inspect, extractedList;
	while(pathList.length > 0) {
		path2Inspect = pathList.pop();
		let stats = yield promisify_lstat(path2Inspect);
		if(stats.isFile()) {
			fileStore.push(path2Inspect);
		}
		else if(stats.isDirectory()) {
			extractedList = yield promisify_readdir(path2Inspect).then(ls=>ls.map(p=>path.resolve(path2Inspect, p))); 
			pathList = pathList.concat(extractedList);
		}
	}
	return fileStore;
});
```
这样改写后，得到了一个从形式上看“循环遍历”文件列表的promise函数，但仅仅是形式而已。我们知道 generator中的yield通常不能返回数据：
```javascript
function *gen() {
	let i = 10;
	i = yield Promise.resolve(i + 1);
	console.log(i);
	return i;
}
let it = gen();
it.next(); //启动
it.next(); // 输出undefined
```
为了能赋值成功，必须把上一次yield的结果作为参数传入本次next中，所以处理promise的写法是
```javascript
it.next().value.then(res=>it.next(res).value);// 输出11
```
co模块则帮我们把上述过程都封装起来，因此本质上还是上面提到的串行promise的版本。从执行速度上看，generator版本要慢很多，在1.6s-1.7s之间，与同步版本差不多，但是不会阻塞event loop。

## 直接改造成stream版本
流，stream，nodejs的核心概念之一。不论是请求、响应、文件还是 socket，这些api的实现都能看到stream的身影。甚至我们平时用的最多的 console.log 打印日志也使用了它。
因此也可以把获取文件列表的功能改造成stream。
```javascript
const fs = require('fs');
const util = require('util');
const path = require('path');
const {Readable} = require('stream');

const promisify_lstat = util.promisify(fs.lstat);
const promisify_readdir = util.promisify(fs.readdir);

class GetFilesStream extends Readable{
	constructor(opts) {
		super(opts);
		if(!opts.rootpath){
			throw new Error('should pass rootpath');
		};
		this._pathList = [opts.rootpath];
	}
	_read() {
		let path2Inspect, extractedList;
		if(this._pathList.length > 0) {
			path2Inspect = this._pathList.pop();
			promisify_lstat(path2Inspect)
			.then((stats)=>{
				if(stats.isFile()) {
					this.push(path2Inspect);
				}
				else if(stats.isDirectory()) {
					promisify_readdir(path2Inspect)
					.then(ls=>ls.map(p=>path.resolve(path2Inspect, p)))
					.then(relativeLS=>{
						if(relativeLS.length>0)
							this._pathList = this._pathList.concat(relativeLS);
						this.push('_directory');
					})
				}	
			});
		}
		else{
			console.log('end');
			this.push(null);
		}
	}
}
```
值得注意的是，可读流只有在push成功后，才能再次调用_read()方法，所以不必担心代码中的promise还没fuifill时就又触发一遍。因此，当读取到的路径是目录时，需要`this.push('_directory')`，否则接下来不会自动调用_read()方法。
由于该用于获取文件列表的stream会在遇到目录路径时输出_directory，所以我们还需要一个transform stream过滤掉这部分不需要的结果。
```javascript
let streamStore = new Transform({
	transform(chunck, enc, next){
		let result;
		if(enc === 'buffer') {
			result = utf8Decoder.write(chunck);
		}
		else{
			result = chunck;
		}
		if(result !== '_directory'){
			this.push(result);
		}
		next();
	}
})
```
所以最终的调用方法是
```javascript
let readSteam = new GetFilesStream({
 	rootpath: '../webpack_demo/node_modules'
});
readSteam.pipe(streamStore).pipe(process.stdout);
```
执行速度方面，stream版本和generator版本区别不大，也是1.6s左右。由于写成stream之后可以直接用pipe接入多种stream，降低内存占用，在需要处理大量文件的场景（如批量上传文件到存储服务器）非常适用。

## 总结
nodejs中的异步io非常高效，利用异步文件api能显著提高处理能力。通过比较promise，gennerator和stream三种方法优化文件列表获取函数，可以发现直接使用promise的版本处理速度是最高的，但有内存占用暴涨的可能。利用generator可以编写简洁明了的异步处理函数，但是由于是串行处理各个路径，速度较慢。stream版本的处理速度和generator版本相似，代码逻辑也比较简单。得益于stream在nodejs中的广泛使用，stream版本更适合需要处理大量文件的场景。

## 参考
1.[Generator 函数的异步应用
](http://es6.ruanyifeng.com/#docs/generator-async)

2.[深入理解 Node.js Stream 内部机制
](http://taobaofed.org/blog/2017/08/31/nodejs-stream/)
