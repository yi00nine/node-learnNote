### Buffer

###### Buffer 结构

- Buffer 是一个像 Array 的对象,主要用于操作字节,且 Buffer 占用的内存不通过 V8 分配,是 node 的 C++层面实现的内存申请.
- 为高效的使用内存,node 采用 slab 分配机制,slab 具有 full 完全分配状态,partial 部分分配状态,empty 没有分配状态

node 以 8KB 来区分 Buffer 是大对象还是小对象
`Buffer.poolSize = 8 * 1024`

###### 分配小 buffer 对象

```js
//Buffer的分配过程
let pool
function(){
  pool = new SlowBuffer(Buffer.poolSize)
  pool.used = 0
}
```

```js
//构造小对象
new Buffer(1024)
```

构造 Buffer 的对象的时候会去检查 pool,判断这个 slab 的剩余空间是否足够,足够的话更新 slab 的分配状态,如果不够会构造新的 slab.

###### 分配大对象

```js
// slab被大对象独占
this.parent = new slowBuffer(this.length)
this.offset = 0
```

###### 字符串转 Buffer

` new Buffer(string,[encoding])`
encoding 不传的时候默认是 Utf-8 格式

###### Buffer 转字符串

` buf.toString([encoding],[start],[end])`

###### Buffer 的拼接

```js
// 这个写法有可能会导致乱码,原因是中文是宽字节,宽字节有可能会被截断.
let fs = require('fs')
let rs = fs.createReadStream(path)
rs.setEncoding('utf-8') //一种解决方法,对Buffer进行解码
let data = ''
rs.on('data', function (chunk) {
  data += chunk
})
rs.on('end', function () {
  console.log(data)
})
```

```js
//正确拼接Buffer
let chunks = []
let size = 0
res.on('data', function (chunk) {
  chunks.push(chunk)
  size += chunk.length
})
res.on('end', function () {
  let buf = Buffer.concat(chunks, size)
  let str = iconv.decode(buf, 'utf-8')
})
```

注:buffer 在网络传输中比 string 的性能要更好
