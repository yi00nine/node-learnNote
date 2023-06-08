### 基础功能

- 请求方法,HTTP_Parser 解析请求报文的时候,将报文头设置为 req.method

```js
function (req,res){
  switch(res.method){
    case 'POST':
      break
    case 'GET':
      break
    case 'PUT':
      break
  }
}
```

- 路径解析,HTTP_Parser 将地址解析为 req.url,匹配到对应的控制器,剩余的值作为参数进行别的判断

```js
function (req,res){
  let pathname = url.parse(req.url).pathname;
  let paths = pathname.split('/');
  let controller = paths[1] || 'index';
  let action = paths[2] || 'index';
  let args = paths.slice(3);
  if (handles[controller] && handles[controller][action]) {
    handles[controller][action].apply(null, [req, res].concat(args));
  } else {
  res.writeHead(500);
  res.end('error');
 }
}
```

- 查询字符串,node 提供了 queryString 来解析 query 参数

```js
let url = require('url')
let queryString = require('querystring')
var query = queryString.parse(url.parse(req.url).query)
```

- Cookie,是一个没有状态的协议,在早期需要 cookie 协助标识和认证用户.

```js
//解析cookie
let parseCookie = function (cookie) {
  let cookies = {}
  if (!cookie) {
    return cookies
  }
  let list = cookie.split(';')
  for (let i = 0; i < list.length; i++) {
    let pair = list[i].split('=')
    cookies[pair[0].trim()] = pair[1]
  }
  return cookies
}
//在业务逻辑执行之前将cookie挂在到req
function(req,res){
  req.cookie = parseCookie(req.headers.cookie)
}
```

    cookie 会造成性能影响,除非 cookie 过期,客户端每一次发的请求都会带上 cookie

- session,cookie 体积过大,且不安全,session 的数据保存在服务器,客户端无法修改相对安全.

```js
//一旦服务器启用了Session,可以约定一个值作为key,一旦服务器监测到用户请求中的cookie没有约定好的key,就会生成这个值
let sessions = {}
let key =  'session_id'
let EXPIRES = 60 * 1000
let generate = () => {
  let session = {}
  session.id = new Date().getTime() + Math.Random()
  session.cookie = {
    expire: new Date().getTime() + EXPIRES,
  }
  sessions[session.id] = session
  return session
}

// 每个请求需要检查cookie是否存在和过期
function(req,res){
  let id = req.cookie[key]
  if(!id){
    req.session = generate()
  }else{
    let session = sessions[id]
    if(session){
      if(session.expire >new Date().getTime()){
        session.cookie.expire = new Date().getTime() + EXPIRES
        req.session = session
      }else{
        delete sessions[id]
        req.session = generate()
      }
    }else{
      req.session = generate()
    }
  }
  handle(req,res)
}

//将新的session设置给客户端,hack writeHeader()
res.writeHeader(){
  let cookies = res.getHeader('Set-Cookie');
  let session = serialize('Set-Cookie', req.session.id);
  cookies = Array.isArray(cookies) ? cookies.concat(session) : [cookies, session];
  res.setHeader('Set-Cookie', cookies);
  return writeHead.apply(this, arguments);
}

```

    session 直接放在对象中,会引起性能问题,目前一般是放在 redis 活着 memcached 中

- http 缓存,一般来说 post,put,delete 请求不需要进行缓存.

```js
// get请求中携带If-Modified-Since: Sun, 03 Feb 2013 06:01:12 GMT
let handle = (req, res) => {
  fs.stat(filename, (err, stat) => {
    let lastModified = stat.mtime.toUTCString()
    if (lastModified === req.header['If-Modified-Since']) {
      res.writeHead(304, 'not Modified')
      res.end()
    } else {
      fs.readFile(filename, (err, data) => {
        let lastModified = stat.mtime.toUTCString()
        res.setHeader('Last-Modified', lastModified)
        res.writeHead(200)
        res.end(data)
      })
    }
  })
}
```

Last-Modified 有两个不是很完善的地方,一是文件的时间改动了但是内容可能没有改动,二是时间戳只能精确到秒级别

```js
// ETag 改进
let handle = (req, res) => {
  fs.readFile(filename, (err, file) => {
    let hash = getHash(file)
    let noneMatch = req.headers['if-none-match']
    if (noneMatch === hash) {
      res.writeHead(304, 'not Modified')
      res.end()
    } else {
      res.setHeader('ETag', hash)
      res.writeHead(200)
      res.end(file)
    }
  })
}
```

使用 ETag 标识客户端还是会发起一次 http 请求,在 http1.0 的时候可以用 expires 告知浏览器缓存的内容和时间,但是由于浏览器和服务器的时间可能会有误差导致错误,http1.1 可以使用 cache-control 来实现相同的功能,max-age 使用类似倒计时的概念,可以忽略客户端和服务器的时间误差

### 数据上传

- node 的 http 模块只对头部进行了解析,内容部分需要手动解析,对于数据量比较大的需要流式的进行解析

```js
let hasBody = function (req) {
  return 'transfer-encoding' in req.headers || 'content-length' in req.headers
}

function (req, res) {
 if (hasBody(req)) {
   let buffers = [];
   req.on('data', function (chunk) {
     buffers.push(chunk);
   });
   req.on('end', function () {
   req.rawBody = Buffer.concat(buffers).toString();
   handle(req, res);
  });
 } else {
   handle(req, res);
 }
}
```

### 路由解析

- 文件路径
- MVC:路由解析根据 Url 找到对应的控制器和行为,行为调用相关的模型进行数据操作,数据操作结束调用相关数据返回客户端
  1. 手动映射:比较灵活,但是工作量大
  2. 自然映射
- RESTful:通过 Url 设计资源,请求方法定义资源的操作,将请求方法也加入到了路由中

### 中间件

`let middleWare = (req,res,next)=>{
  //todo
  next()
}`

```js
app.use = (path) => {
  let handle = {
    path: pathRegexp(path),
    stack: Array.prototype.slice.call(arguments, 1),
  }
  routes.all.push(handle)
}

let match = (pathName, routes) => {
  for(let i =0; i<routes.length; i++;){
    let route = routes[i]
    let reg = route.path.regexp
    let matched = reg.exec(pathname)
    if(matched){
      handle(req,res,route,stack)
      return true
    }
  }
  return false
}

let handle = (req,res,stack)=>{
  let next =()=>{
    let middleware = stack.shift()
    if(middleware){
      middleware(req,res,next)
    }
  }
  next()
}

```

异常处理,中间件中产生的异常需要自己传递出来

```js
//
let handle = (req, res, stack) => {
  let next = (err) => {
    if (err) {
      return handle(err, req, res, stack)
    }
    let middleware = stack.shift()
    if (middleware) {
      try {
        middleware(req, res, next)
      } catch {
        next(err)
      }
    }
  }
  next()
}
```
