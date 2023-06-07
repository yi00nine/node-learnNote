### node.js 结构

- node.js 标准库:由 javascript 编写的应用程序调用 api
- node bindings:上层 javascript 通过 node bindings 调用 c++
- 由 c++实现的底层, V8:为 JS 提供了非浏览器端的运行环境
  libuv:关键组成部分,提供跨平台,异步 io,事件驱动,线程池等能力
  C-ares:异步处理 dns

### node.js 特点

- 异步 I/O:在 node 中绝大部分的操作都是异步方式进行调用的
- 单线程:node 是单线程指的是 js 的应用程序是单线程的,但是底层的 I/O 操作是由 libuv 实现的,因此 node 具有单线程 节省创建线程,节省线程上下文切换开销等优点.以及异步 IO 可以更好的利用 cpu.
- 回调和事件驱动:
  node 会执行一个类似于 while(true)的循环,每执行一次循环就是一次 Tick,每一个 Tick 的观察者会查看是否有事件需要进行处理.如果有就取出事件以及事件的回调函数.

### 异步 I/O

```
具体流程(以fs.open为例):
  fs.open()通过指定的路径和参数打开一个文件,获取到这个文件的描述符,然后js通过nodebindings去调用node file.cc,c++通过libuv跨平台调用uv_fs_open()方法,在调用的过程中内核会产生FSWrapReq请求对象,将js传入的参数以及当前的方法和回调函数都设置在oncomplete_sym,封装在FSWrapReq这个对象中.对象封装完毕之后,调用QueneUserWorkItem方法将这个对象放到线程池进行等待,执行完毕之后将结果放在req.result,然后通过IOPC告知这个对象已经操作完成,并将线程归还线程池,事件循环I/O观察者在每次Tick的时候都会通过IOPC来检查线程中是否有执行完毕的请求,如果有,会把存在的请求对象加入到I/O观察者队列,I/O观察者会去取出请求对象的result以及oncomplete_sym中的呼叫函数进行执行
```

### 异步 I/O 总结

- node 发起异步请求
- 封装请求对象,设置请求参数以及回调函数
- 请求对象放入线程池等待执行
- 线程池执行请求对象的 IO 操作
- 将执行完成的结果放在请求的对象中
- 通知 IOCP 执行结束,线程归还线程池
- 事件循环观察者获取请求对象,取出回调函数并执行
