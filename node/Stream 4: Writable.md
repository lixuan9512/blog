# Writable

## 什么是可写流

可写流是对数据流向设备的抽象，用来消费上游流过来的数据，通过可写流程序可以把数据写入设备，常见的是本地磁盘文件或者 TCP、HTTP 等网络响应。

看一个之前用过的例子

```javascript
process.stdin.pipe(process.stdout);
```

**\*process.stdout*** 是一个可写流，程序把可读流 process.stdin 传过来的数据写入的标准输出设备。在了解了可读流的基础上理解可写流非常简单，流就是有方向的数据，其中可读流是数据源，可写流是目的地，中间的管道环节是双向流。

## 可写流使用

调用可写流实例的 **write() **方法就可以把数据写入可写流

```javascript
const fs = require('fs');
const rs = fs.createReadStream('./w.js');
const ws = fs.createWriteStream('./copy.js');

rs.setEncoding('utf-8');
rs.on('data', chunk => {
  ws.write(chunk);
});
```

前面提到过监听了可读流的 data 事件就会使可读流进入流动模式，我们在回调事件里调用了可写流的 write() 方法，这样数据就被写入了可写流抽象的设备中，也就是当前目录下的 copy.js 文件。

write() 方法有三个参数

- **chunk** {String| Buffer}，表示要写入的数据
- **encoding** 当写入的数据是字符串的时候可以设置编码
- **callback** 数据被写入之后的回调函数

## 自定义可写流

和自定义可读流类似，简单的自定义可写流只需要两步

1. 继承 stream 模块的 **Writable** 类
2. 实现 **_write()** 方法

我们来实现一个简单的可写流，把传入可写流的数据转成大写之后输出到标准输出设备（比较好的例子可能是写入本地磁盘文件，但涉及过多的 fs 操作，比较麻烦，偷个懒。写入标准输出设备也是一种写入行为）

```javascript
const Writable = require('stream').Writable

class OutputStream extends Writable {
    _write(chunk, enc, done) {
        // 转大写之后写入标准输出设备
        process.stdout.write(chunk.toString().toUpperCase());
        // 此处不严谨，应该是监听写完之后才调用 done
        process.nextTick(done);
    }
}

module.exports = OutputStream;
```

和最终可写流暴露出来的 write() 方法一样， _write() 方法有三个参数，作用类似

- **chunk** 写入的数据，大部分时候是 buffer，除非 decodeStrings 被设置为 false
- **encoding** 如果数据是字符串，可以设置编码，buffer 或者 object 模式会忽略
- **callback** 数据写入后的回调函数，可以通知流传入下一个数据；当出现错误的时候也可以设置一个 error 参数

当然其实还有一个 _writev() 方法可以实现，这个方法仅被滞留的写入队列调用，可以不实现。

## 实例化可写流

有了可写流的类之后我们可以实例化使用了，实例化可写流的时候有几个 option 可选，了解一下可以帮助我们理解后面要用的知识

- **objectMode** <Boolean> 默认是 false， 设置成 true 后 writable.write() 方法除了写入 string 和 buffer 外，还可以写入任意 JavaScript 对象。很有用的一个选项，后面介绍 transform 流的时候详细介绍
- **highWaterMark** <Number> 每次最多写入的数据量， Buffer 的时候默认值 16kb， objectMode 时默认值 16
- **decodeStrings** <Boolean> 是否把传入的数据转成 Buffer，默认是 true

这样我们就更清楚的知道 _write() 方法传入的参数的含义了，而且对后面介绍 back pressure 机制的理解很有帮助。

## 事件

和可读流一样，可写流也有几个常用的事件，有了可读流的基础，理解起来比较简单

> - **pipe** 当可读流调用 pipe() 方法向可写流传输数据的时候会触发可写流的 pipe 事件
> - **unpipe** 当可读流调用 unpipe() 方法移除数据传递的时候会触发可写流的 unpipe 事件

这两个事件用于通知可写流数据将要到来和将要被切断，在通常情况下使用的很少。



*writeable.write()* 方法是有一个 bool 的返回值的，前面提到了 **highWaterMark**，当要求写入的数据大于可写流的 highWaterMark 的时候，数据不会被一次写入，有一部分数据被滞留，这时候 writeable.write() 就会返回 **false**，如果可以处理完就会返回 true

> **drain** 当之前存在滞留数据，也就是 writeable.write() 返回过 false，经过一段时间的消化，处理完了积压数据，可以继续写入新数据的时候触发（drain 的本意即为排水、枯竭，挺形象的）



除了 write() 方法可写流还有一个常用的方法 end()，参数和 write() 方法相同，但也可以不传入参数，表示没有其它数据需要写入，可写流可以关闭了。

> **finish** 当调用 writable.end() 方法，并且所有数据都被写入底层后会触发 finish 事件



同样出现错误后会触发 **error** 事件

## back pressure

了解了这些事件，结合上之前提到的可读流的一些知识，我们就能探讨一些有意思的话题了。在最开始我们提到过用流相对于直接操作文件的好处之一是不会把内存压爆，那么流是怎么做到的呢？

最开始我们可能会想到因为流不是一次性把所有数据载入内存处理，而是一边读一边写。但我们知道一般读取的速度会远远快于写入的速度，那么 pipe()  方法是怎么做到供需平衡的呢？

回忆一些基础知识，我们自己来实现一下 pipe() 方法的核心原理

1. 可读流有流动和暂停两种模式，可以通过 **pause() **和** resume() **方法切换
2. 可写流的 **write() **方法会返回是否能处理当前的数据，每次可以处理多少是 **hignWatermark** 决定的
3. 当可写流处理完了积压数据会触发 **drain** 事件

我们可以利用这三点来做到数据读取和写入的同步，还是使用之前的例子，但为了使消费速度降下来，我们各一秒再通知完成

```javascript
class OutputStream extends Writable {
    _write(chunk, enc, done) {
        // 转大写之后写入标准输出设备
        process.stdout.write(chunk.toString().toUpperCase());
        // 故意延缓通知继续传递数据的时间，造成写入速度慢的现象
        setTimeout(done, 1000);
    }
}
```

我们使用一下自定义的两个类

```javascript
const RandomNumberStream = require('./RandomNumberStream');
const OutputStream = require('./OutputStream');

const rns = new RandomNumberStream(100);
const os = new OutputStream({
    highWaterMark: 8 // 把水位降低，默认16k还是挺大的
});

rns.on('data', chunk => {
    // 当待处理队列大于 highWaterMark 时返回 false
    if (os.write(chunk) === false) { 
        console.log('pause');
        rns.pause(); // 暂停数据读取
    }
});

// 当待处理队列小于 highWaterMark 时触发 drain 事件
os.on('drain', () => {
    console.log('drain')
    rns.resume(); // 恢复数据读取
});
```

结合前面的三点和注释很容易看懂上面代码，这就是 pipe() 方法起作用的核心原理。数据的来源的去向我们有了大概了解，后面可以开始介绍数据的加工

- duplex
- transform

​