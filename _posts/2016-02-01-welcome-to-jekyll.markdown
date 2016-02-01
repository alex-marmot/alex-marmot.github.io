---
layout: post
title:  "查找 Nodejs 中内存泄漏的简易指南"
date:   2016-02-01 12:07:14 +0800
categories: translation
---
# 查找 Nodejs 中内存泄漏的简易指南

## 感谢作者 Alex Kras
## [原文链接](http://www.alexkras.com/simple-guide-to-finding-a-javascript-memory-leak-in-node-js)

### 内容大纲:
- 简介
- 基本理论
- 步骤1 重现并确认问题
- 步骤2 至少获取三个堆转储文件
- 步骤3 找到问题
- 步骤4 确认问题是否被解决
- 其他资源的链接
- 总结


## 简介

几个月前，在我调试 Node.js 中的一个内存泄漏问题。我发现专注于此的文章相当多。但在仔细阅读过其中一部分之后，我迷茫依旧。

我希望这篇博文能成为大家提供一个简单的指引。我会概述一个简单易行的方法，(我认为)它应该能成为排查 Node 中任何内存泄漏问题的起始点。对于一些案例而言,这个方法也许不够用。我会
提供链接以兹参考。

## 基本理论

Javascript 是一门具有垃圾回收的语言。因此,所有 Node 程序使用的内存
都由 V8 引擎自动分配和回收。

V8 引擎是如何知道何时回收内存的? 它会从根节点开始追踪, 将程序中所有的变量都维护在 Memory Graph 中。Javascript 中有四种数据类型: 布尔, 字符串, 数字以及对象。前三者都是简单的类型并且它们只能持有那些被赋值给它们的数据(例如 字符串的文本)。对象, JavaScript 中除了前三个类型之外都是对象(比如说 Arrays 就是对象。), 能保持其他对象的引用(指针)。

![memory-graph](http://www.alexkras.com/wp-content/uploads/memory-graph.png)

V8 引擎会周期性的校验 Memory Graph，尝试鉴别出那些已不能被根节点(root node)访问到的数据。
如果那些数据不能被根节点(root node)访问到, V8 引擎就会认为它们已不再被使用而释放了那些内存。这一过程称之为 **垃圾回收**。

译者注:
  许多主流程序语言中（如Java、C#、Lisp），都是使用可达性分析来判定对象是否存活的。这个算法的基本思路就是通过一系列的称为GC根节点（GC Roots）的对象作为起始点，从这些节点开始进行向下搜索，搜索所走过的路径成为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连（用图论的话来说就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。如图1所示，对象object 5、object 6、object 7虽然互相有关联，它们的引用并不为0，但是它们到GC Roots是不可达的，因此它们将会被判定为是可回收的对象。
  -- 引用自 [http://www.infoq.com/cn/articles/jvm-memory-collection](http://www.infoq.com/cn/articles/jvm-memory-collection)

#### 内存泄漏什么时候发生?

在 JavaScript 中,当一些不再使用的数据依然能被根节点(root node)访问时就会发生内存泄漏。
V8 引擎会认为那些数据仍然在被使用而不会释放相关的内存。**为了能调试内存泄漏问题, 我们需要定位到那些异常数据并确保 V8 引擎能够清除它们。**

另外你需要记得的重要一点是垃圾回收不是一直运行的。通常 V8 引擎会在它认为恰当的时候触发垃圾回收。比如说它能够周期性的运行垃圾回收，也可以在它认为内存越来越少时,鲁莽地运行垃圾回收。Node 对每个进程能使用的内存数有上限要求。所以 V8 引擎 必须善加利用它所拥有的。

![node-error](http://www.alexkras.com/wp-content/uploads/node-error.png)


而**后一种垃圾回收**可能会成为**性能显著下降的源头**。

想象一下，你有一个大量内存泄漏的应用。 很快, Node 进程就会用完分配给它的内存, 这会导致 V8 触发不合时宜地垃圾回收。但是,由于大部分的数据仍然是可被访问的状态, 所以只有非常少的的内存会被回收, 大部分的应该被回收的内存仍然会被保留。

很快, Node 进程会再次耗尽内存。这会导致触发另一次的垃圾回收。在你意识到之前, 你的应用早已为了维持进程的正常功能而不得不频繁地进行垃圾回收。既然 V8 引擎花费大量时间处理垃圾回收,那么只有少的可怜的资源用于真正的程序上。


## 步骤一: 重现并确认问题

如前所述 V8 引擎自身有着复杂的逻辑来判断合适触发垃圾回收的时机。 将此谨记于心: 既然我们看到一个 Node 进程的内存占用量持续上升, **在我们不能确定那是内存泄漏, 直到我们确认垃圾回收已经在执行**清理那些未被使用的内存了。

幸运的是, Node 允许我们手动触发垃圾回收, 这也是我们试图确认内存泄漏时要做的第一件事。我们可以通过在运行 Node 程序时, 添加标识 `--expose-gc` (i.e. `node --expose-gc index.js`)。一旦 Node 以此种模式运行, 你就可以在任何时候通过在代码中调用 `global.gc()` 来触发垃圾回收。

你还可以通过调用 `process.memoryUsage().heapUsed` 来检查你的进程所消耗的内存总量。

**通过手动触发垃圾回收及检查所消耗的堆栈情况, 你能够确定程序中是否有内存泄漏。**

#### 举个栗子

我创建了一个内存泄漏的样例: [https://github.com/akras14/memory-leak-example](https://github.com/akras14/memory-leak-example)

你可以先 clone 它, 然后运行 `npm install`, 最后运行 `node --expose-gc index.js` 来实践检验。

``` javascript
"use strict";
require('heapdump');

var leakyData = [];
var nonLeakyData = [];

class SimpleClass {
  constructor(text){
    this.text = text;
  }
}

function cleanUpData(dataStore, randomObject){
  var objectIndex = dataStore.indexOf(randomObject);
  dataStore.splice(objectIndex, 1);
}

function getAndStoreRandomData(){
  var randomData = Math.random().toString();
  var randomObject = new SimpleClass(randomData);

  leakyData.push(randomObject);
  nonLeakyData.push(randomObject);

  // cleanUpData(leakyData, randomObject); //<-- Forgot to clean up
  cleanUpData(nonLeakyData, randomObject);
}

function generateHeapDumpAndStats(){
  //1. Force garbage collection every time this function is called
  try {
    global.gc();
  } catch (e) {
    console.log("You must run program with 'node --expose-gc index.js' or 'npm start'");
    process.exit();
  }

  //2. Output Heap stats
  var heapUsed = process.memoryUsage().heapUsed;
  console.log("Program is using " + heapUsed + " bytes of Heap.")

  //3. Get Heap dump
  process.kill(process.pid, 'SIGUSR2');
}

//Kick off the program
setInterval(getAndStoreRandomData, 5); //Add random data every 5 milliseconds
setInterval(generateHeapDumpAndStats, 2000); //Do garbage collection and heap dump every 2 seconds
```

这段程序会:

  1. 每过 5 毫秒 生成一个随机数对象并将之存入两个数组中。一个叫 leakyData, 另一个叫 nonLeakyData。我们每过 5 毫秒 会清空 nonLeakyData 而“忘记”清理 leakyData。
  2. 每过两秒程序会内存使用总量(并且还会生成一个当前堆状态的堆转储文件, 我们将在下一小节进一步讨论)。

如果你以 `node --expose-gc index.js` (或者 `npm start`)的方式运行程序, 它将会输出内存统计信息。让它运行一两分钟然后用 `Ctr + c` 干掉它。

即使我们每 2 毫秒 触发一次垃圾回收, 你将会看到内存消耗的快速增长。下面就是我们得到的统计数据:

``` javascript
//1. Force garbage collection every time this function is called
try {
  global.gc();
} catch (e) {
  console.log("You must run program with 'node --expose-gc index.js' or 'npm start'");
  process.exit();
}

//2. Output Heap stats
var heapUsed = process.memoryUsage().heapUsed;
console.log("Program is using " + heapUsed + " bytes of Heap.")
```

统计数据类似于下面这样的:

``` javascript
Program is using 3783656 bytes of Heap.
Program is using 3919520 bytes of Heap.
Program is using 3849976 bytes of Heap.
Program is using 3881480 bytes of Heap.
Program is using 3907608 bytes of Heap.
Program is using 3941752 bytes of Heap.
Program is using 3968136 bytes of Heap.
Program is using 3994504 bytes of Heap.
Program is using 4032400 bytes of Heap.
Program is using 4058464 bytes of Heap.
Program is using 4084656 bytes of Heap.
Program is using 4111128 bytes of Heap.
Program is using 4137336 bytes of Heap.
Program is using 4181240 bytes of Heap.
Program is using 4207304 bytes of Heap.
```

如果你将数据绘制成图, 那样内存消耗的增长会更加显著。

![with-memory-leak](http://www.alexkras.com/wp-content/uploads/with-memory-leak.png)

注意: 如果你好奇我是如何绘制数据的,请继续阅读, 不然就直接跳到下一小节。

我先将输出的统计数据存入一个 JSON 文件中, 然后我用一小段 Python 代码来读取并绘制。我单独为此建立了一个分支以避免混淆, 你可以在此处检出: [https://github.com/akras14/memory-leak-example/tree/plot](https://github.com/akras14/memory-leak-example/tree/plot)

相关部分代码在此:
``` javascript
var fs = require('fs');
var stats = [];

//--- skip ---

var heapUsed = process.memoryUsage().heapUsed;
stats.push(heapUsed);

//--- skip ---

//On ctrl+c save the stats and exit
process.on('SIGINT', function(){
  var data = JSON.stringify(stats);
  fs.writeFile("stats.json", data, function(err) {
    if(err) {
      console.log(err);
    } else {
      console.log("\nSaved stats to stats.json");
    }
    process.exit();
  });
});
```

和

``` python
#!/usr/bin/env python

import matplotlib.pyplot as plt
import json

statsFile = open('stats.json', 'r')
heapSizes = json.load(statsFile)

print('Plotting %s' % ', '.join(map(str, heapSizes)))

plt.plot(heapSizes)
plt.ylabel('Heap Size')
plt.show()

```

你可以检出 polt 分支, 像之前一样运行程序。一旦你完成程序运行,就执行 `python plot.py` 来绘图。要运行相关的 Python 脚本, 你需要先安装好代码库 Matplotlib。

或者你可以将数据导入 Excel 中来制作图表。

## 步骤2 至少获取三个获取堆转储文件

好了, 我们已经重现了问题, 那现在怎么做呢? 现在我们需要找出问题的所在并进行修复 :)

你可能已经注意到了我样例代码中的那么几行:

``` javascript
require('heapdump');
// ---skip---

//3. Get Heap dump
process.kill(process.pid, 'SIGUSR2');

// ---skip---
```

我使用的是 node-heapdump 模块, 你能在这儿找到相关信息:

[https://github.com/bnoordhuis/node-heapdump](https://github.com/bnoordhuis/node-heapdump)

为了能使用 node-heapdump, 你需要这么做:

1. 安装
2. 在程序顶部导入
3. 在类 Unix 平台上调用 `kill -USR2 {{pid}}`

如果你之前没见过 `kill` 部分, 现在给你科普下这是个 Unix 环境下的命令, 它允许你(或其他人)向任何正在运行的进程发送一个自定义的信号(亦称 用户信号)。
 Node-heapdump 被配置成获取该进程中的 heap dump, 它会在任何时候接收 **user signal two** 也就是 `-USR2`, 进程 id 紧跟其后。

在栗子中, 通过运行 `process.kill(process.pid, 'SIGUSR2')` 来自动执行 `kill -USR2 {{pid}}`。 `process.kill` 是 Node 对 `kill` 命令的包装, `SIGUSR2` 是 Node 中表示 `-USR2` 的方式
, `process.pid` 则会获取当前 Node 进程的 id。每次垃圾回收执行后，我都会执行此命令获取一个干净的堆转储文件。

我不认为 `process.kill(process.pid, 'SIGUSR2')` 能在 Windows 上生效, 但是你可以用 `heapdump.writeSnapshot()` 来代替。

可能最初使用 `heapdump.writeSnapshot()` 会稍微容易点, 但是我想说的是在类 Unix 平台上你通过 `kill -USR2 {{pid}}` 命令来获取堆转储文件能更方便。

下一节我们将论及如何使用生成的堆转储文件来分离出内存泄漏的地方。

## 步骤3 找到问题

在第二步时, 我们生成了一堆堆转储文件。我们至少需要三个, 很快你就会知道原因了。

一旦你有了堆转储文件, 打开 Chrome 浏览器中的开发者工具(Windows 上使用 F12, Mac 上使用 Command + Options + i)。

在开发者工具中选择 “Profiles” 标签页, 选择位于底部的 “Load” 按钮, 选中你选中的第一个堆转储文件, 其内容将会被载入到视图中, 显示如下:

![1st-Heap-Dump](http://www.alexkras.com/wp-content/uploads/1st-Heap-Dump.png)

接着再载入两个堆转储文件到视图中。 例如, 你可以使用你所获取的最后两个堆转储文件。最重要的是必须以获取时的顺序载入。你的 "Profiles" 标签页显示的内容类似于下图。

![3-Heap-Dumps](http://www.alexkras.com/wp-content/uploads/3-Heap-Dumps.png)

如上图所示, 堆转储文件每次都会变大一点。

#### 3 堆转储文件大法

一旦你载入了堆转储文件, 你将会看到一大坨的子视图, 你很容易迷失其中。 但是, 我发现一个视图特别有用。

点击你最后获取的那个堆转储文件, 它会立即显示 “总览” 视图(Summary). 如下图所示的, 你能在 “总览” 旁看到一个下拉列表上面显示着 “All”。点击它能够选择相应的堆转储文件。

![3-Heap-Dump-View](http://www.alexkras.com/wp-content/uploads/3-Heap-Dump-View.png)

它会向你展示出所有在你三个堆转储文件相应时间段内某刻被分配的对象。它们就是你的重点关注调查对象, 因为它们应该已经被垃圾回收清理了。

这东西相当酷吧, 但它很难被找到所以很容易被忽略掉。

#### 至少在一开始忽略诸如字符串之类的对象

在样例程序中完成了上述内容后, 我以接下来的内容的结束。

注意 shallow size 代表了对象本身的大小, 而 retained size 包含了该对象及其子对象在内的大小。

![memory-leak](http://www.alexkras.com/wp-content/uploads/memory-leak.png)

我的最后截屏显示有五个不该存在的实体: 数组(array), 编译后的代码 (compiled code), 字符串(string), 系统对象(system), 和 SimpleClass。

在这些实体中, 只有 SimpleClass 看着眼熟, 因为它来自样例程序。

```javascript
var randomObject = new SimpleClass(randomData);
```

从数组 (array) 或字符串 (string) 开始查起比较诱人。在总览视图(Summary view)中, 所有的对象都按照它们构造器的名字分组。 在数组或字符串的例子中, 它们都是 Javascript 引擎的内部构造器。
 所以当你的程序在这些对象中持有数据, 你会被干扰难以定位泄漏源。

这就是为什么要在一开始先跳过这些, 而是看看你能否找到其他更可疑的对象,像是样例程序中的 **SimpleClass** 的构造器。

点击 SimpleClass 构造器旁的箭头并选择结果列表中任何对象就会在下一行显示其构成(如上图所示)。这样你就能非常方便的追查出 leakyData 数组持有着我们的数据。

如果你不像我这般幸运, 你可能就需要查看内部构造器了(the internal constructors )(比如说 字符串)以找出内存泄露源。那样的话,尝试着找出一些在内部构造器分组(internal constructors groups)中经常出现的的分组的值。
以其为提示,能助你定位内存泄漏源。

例如, 在样例程序中,你可能会观察到许多看似由随机数转换而成的字符串。如果你检测它们的 retainer paths, Chrome 的开发者工具就会指向 leakyData 数组。

## 步骤4 确认问题是否被解决

在你确认并解决了一个疑似内存泄漏问题后, 你应该能发现堆栈使用情况的巨大变化。

如果我们去掉样例程序中下面这行的注释:

``` javascript
    cleanUpData(leakyData, randomObject); //<-- Forgot to clean up
```

 然后再次像第一步时那样运行应用并好好观察接下来的输出结果:

 ``` javascript

Program is using 3756664 bytes of Heap.
Program is using 3862504 bytes of Heap.
Program is using 3763208 bytes of Heap.
Program is using 3763400 bytes of Heap.
Program is using 3763424 bytes of Heap.
Program is using 3763448 bytes of Heap.
Program is using 3763472 bytes of Heap.
Program is using 3763496 bytes of Heap.
Program is using 3763784 bytes of Heap.
Program is using 3763808 bytes of Heap.
Program is using 3763832 bytes of Heap.
Program is using 3758368 bytes of Heap.
Program is using 3758368 bytes of Heap.
Program is using 3758368 bytes of Heap.
Program is using 3758368 bytes of Heap.
 ```

如果我们还将结果绘制成图, 显示结果可能跟下面的相似:

![without-memory-leak](http://www.alexkras.com/wp-content/uploads/without-memory-leak.png)

万岁, 内存泄漏被干掉啦。

需要注意的是, 起始时的一段高内存占用仍然存在直到程序稳定时才正常。小心那个高峰, 确保你没有把它当做内存泄漏。

## 其他资源的链接

#### 利用 Chrome 的开发者工具来概览内存情况

[Memory Profiling with Chrome DevTools](https://youtu.be/L3ugr9BJqIs)

你在本文中看到的内容都源自该视频。这篇文章存在的唯一原因是, 我不得不在两周内看三遍该视频才能抓住重点(我认为是), 我希望别人能能快得完成这样的发现之旅。

强烈推荐这个视频作为本文的补充。

#### 另一个有用的工具 – memwatch-next

这是我认为另一个值得一提的工具是。你可以在[这儿](https://hacks.mozilla.org/2012/11/tracking-down-memory-leaks-in-node-js-a-node-js-holiday-season/)知道更多的理由(精简版, 能节省你的时间)。

或者直接访问代码库: [https://github.com/marcominetti/node-memwatch](https://github.com/marcominetti/node-memwatch)

方便你偷懒, 你可以通过 `npm install memwatch-next` 来进行安装。

通过两个事件来使用它:
``` javascript
var memwatch = require('memwatch-next');
memwatch.on('leak', function(info) { /*Log memory leak info, runs when memory leak is detected */ });
memwatch.on('stats', function(stats) { /*Log memory stats, runs when V8 does Garbage Collection*/ });

//It can also do this...
var hd = new memwatch.HeapDiff();
// Do something that might leak memory
var diff = hd.end();
console.log(diff);
```

控制台最后输出的类似于下面的内容, 向你展示了哪种类型的对象的内存占用中不断增长。
``` javascript
{
  "before": { "nodes": 11625, "size_bytes": 1869904, "size": "1.78 mb" },
  "after":  { "nodes": 21435, "size_bytes": 2119136, "size": "2.02 mb" },
  "change": { "size_bytes": 249232, "size": "243.39 kb", "freed_nodes": 197,
    "allocated_nodes": 10007,
    "details": [
      { "what": "String",
        "size_bytes": -2120,  "size": "-2.07 kb",  "+": 3,    "-": 62
      },
      { "what": "Array",
        "size_bytes": 66687,  "size": "65.13 kb",  "+": 4,    "-": 78
      },
      { "what": "LeakingClass",
        "size_bytes": 239952, "size": "234.33 kb", "+": 9998, "-": 0
      }
    ]
  }
}
```

很酷吧。

#### 来自 developer.chrome.com 的 JavaScript 内存使用分析

[https://developer.chrome.com/devtools/docs/javascript-memory-profiling](https://developer.chrome.com/devtools/docs/javascript-memory-profiling)

绝对值得一看。 它包含了所有我论及的内容并更深入更详细更准确。

不要忽视 Addy Osmani 最后的谈话, 他在那儿提及了很多关于 debugging 的贴士和资源。

[the video](https://youtu.be/LaxbdIyBkL0)

你可以在这儿获取 [slide](https://speakerdeck.com/addyosmani/javascript-memory-management-masterclass) 和 [源码](https://github.com/addyosmani/memory-mysteries):

## 总结

1. 在尝试重现并定位内存泄漏时, 手动触发垃圾回收。你可以通过在程序中调用 `global.gc()` 并并在运行时添加 `--expose-gc` 标识。
2. 通过使用 [https://github.com/bnoordhuis/node-heapdump](https://github.com/bnoordhuis/node-heapdump) 至少获取 3 个 Heap Dumps
3. 使用 3 heap dump 大法来隔离出内存泄漏的部分
4. 确认内存泄漏已被解决
5. 享受解决后获得的好处
