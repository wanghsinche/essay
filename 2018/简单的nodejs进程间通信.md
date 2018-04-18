# 简单的nodejs进程间通信.md

>　组里决定将原有的fis3发布平台改造成基于npm包管理系统的构建发布平台。出于安全性和解耦的考虑，很有必要把平台分成两部分：1、供前端调用的API服务器，2、真正的构建发布程序。前端发送构建请求到API服务器，经过一系列的鉴权，验证和数据库操作，API服务器向构建发布程序发送指令并返回成功代码。发布任务转由构建发布程序完成，并用websocket实时输出日志到客户端。所以必须使用一种合适的IPC连接这两个子系统。

## IPC介绍
IPC有管道、消息队列、套接字、共享内存和信号量几种。其中信号量用于控制资源的访问，通常与其它IPC方法配合使用。

### 管道
管道提供了类似nodejs中stream机制。读进程从首部读取数据，写进程从末尾不断写入数据。如果缓冲区为空，读进程将被阻塞，相反的，当缓冲区充满时，写进程无法继续写入，直到读进程将数据读出，缓冲区重新可用。管道有三种：1、普通管道只能从父进程单向流向子进程；2、流管道可以在父子进程间传递消息；3、命名管道没有以上两种管道的限制，可以在两个不同进程间相互传递消息。

1.*消息队列*
管道只能传递字节流，而且缓冲区较小。消息队列克服以上缺点，可以传递有格式的数据，而且接收者在消息到达很长时间后再取用，非常适合并发较高而处理能力不足的场景。

2.*套接字*
套接字socket功能类似命名管道，而且允许不同机器间的进程通信。使用相当广泛

3.*共享内存*
共享内存是最快的IPC方式。操作系统建立一块共享内存，并将其映射到每个进程的地址空间上，各个进程就可以直接对这块共享内存进行读写，实现通信。共享内存不提供同步机制，需要开发者自己实现，这点比较麻烦。

## nodejs中常用的IPC
1.*child_process.fork*
child_process.fork() 方法是 child_process.spawn() 的一个特殊情况，专门用于衍生新的 Node.js 进程，返回一个 ChildProcess 对象。 返回的 ChildProcess 会有一个额外的内置的通信通道，它允许消息在父进程和子进程之间来回传递。 官网的例子：
父进程：
```javascript
const cp = require('child_process');
const n = cp.fork(`${__dirname}/sub.js`);

n.on('message', (m) => {
  console.log('父进程收到消息：', m);
});

n.send({ hello: 'world' });
```
子进程
```javascript
process.on('message', (m) => {
  console.log('子进程收到消息：', m);
});

process.send({ foo: 'bar', baz: NaN });
```
利用这种方式，需要把API服务器，或者发布程序做成父进程，另一个成为子进程。或者都作为子进程，由父进程中转消息。不利于解耦和调试，也不能避免父进程出错导致所有服务都不可用的情况。而且API服务器和发布程序也必须运行在同一个机器上。

2.*nodejs+redis*
redis非常常用，利用自带的list类型，可以得到一个简陋的消息队列。API服务器把发布任务rpush到list中，发布程序在空闲时从list中取出发布任务进行处理。由于websocket由API服务器建立和管理，所以发布程序还需要不断将发布任务的日志通知API服务器，再通过相应的websocket发送给客户端。正好redis提供了订阅/发布功能，发布程序可以将日志发布给API服务器使用。

3.*net模块*
用nodejs的net模块，在API服务器和发布程序间建立socket非常简单。需要注意的地方是socket只能传送无格式字节流，需要自己实现解包和消息封装。
首先需要做一个解包和消息封装的基类，继承自EventEmitter，这里选用‘\n’为分隔符，并在发送消息时用json.stringify对消息转义，防止消息本身带有‘/n’：
```javascript
/* socket 解包器 */
/* 约定数据帧以\n结尾,并且都是json encode */
class RawDataParser extends EventEmitter {
    constructor() {
        super()
        this.sep = '\n'
        this.rawStringBuffer = []
    }
    encode(message) {
        return JSON.stringify(message) + '\n';
    }
    feed(chunk) {
        const str = chunk.toString('utf8')
        const sepPos = str.indexOf(this.sep)
        if (sepPos > -1) {
            this.rawStringBuffer.push(str.substring(0, sepPos))
            let tempStr = this.rawStringBuffer.join('')
            this.rawStringBuffer = [str.substring(sepPos + 1)]
            this.emit('message', tempStr)
        }
        else {
            this.rawStringBuffer.push(str)
        }
    }
}
```
接着在这个解包器基础上编写socket 客户端，并提供重连，send和message等方法和事件。
```JavaScript

/* IPC socket 客户端 */
class IPCClient extends RawDataParser {
    constructor(option) {
        super()
        this.connected = false
        if (typeof option === 'number') {
            this.socketPort = option
            this.connection = net.createConnection(this.socketPort, () => {
                this.connecthandle()
            })
        }
        else if (option instanceof net.Socket) {
            this.connected = true
            this.connection = option
        }
        else {
            throw new Error('option should be a sock or port')
        }
        this.bindEvent()
    }
    reConnect() {
        if (this.socketPort) {
            this.connection = net.createConnection(this.socketPort, () => {
                this.connecthandle()
            })
            this.bindEvent()
        }
    }
    bindEvent() {
        this.connection.on('connect', () => {
            this.emit('connect')
        })
        this.connection.on('end', () => {
            this.connected = false
            this.emit('end')
        })
        this.connection.on('error', err => {
            this.connected = false
            this.emit('error', err)
        })
        this.connection.on('data', chunk => {
            this.feed(chunk)
        })
    }
    connecthandle() {
        this.connected = true
    }
    send(msg, ...others) {
        this.connection.write(this.encode(msg), ...others)
    }
}
```


目前构建发布系统需要的功能很简单，所以采用了这种方式。






