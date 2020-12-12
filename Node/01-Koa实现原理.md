### 知识点

#### Koa

- 概述：Koa 是⼀个新的 **web 框架**， 致力于成为 **web 应用**和 **API 开发**领域中的⼀个更小、更富有表现力、更健壮的基石。

  - Koa是Express的下⼀代基于Node.js的web框架
  - Koa2完全使用Promise并配合 async 来实现异步

- 特点：

  - 轻量，无捆绑
  - 中间件架构
  - 优雅的API设计
  - 增强的错误处理

- 中间件机制、请求、响应处理

  ```js
  const Koa = require('koa')
  const app = new Koa()
  
  app.use((ctx, next) => {
    ctx.body = [
      {
        name: 'tom'
      }
    ]
    next()
  })
  
  app.use((ctx, next) => {
    // ctx.body && ctx.body.push(
    // {
    // name:'jerry'
    // }
    // )
    console.log('url' + ctx.url)
    if (ctx.url === '/html') {
      ctx.type = 'text/html;charset=utf-8'
      ctx.body = `<b>我的名字是:${ctx.body[0].name}</b>`
    }
  })
  
  app.listen(3000)
  ```

- Koa中间件机制：Koa中间件机制就是函数式 组合概念 Compose的概念，将⼀组需要顺序执行的函数复合为⼀个函数，外层函数的参数实际是内层函数的返回值。洋葱圈模型可以形象表示这种机制，是[源码](https://github.com/koajs/compose#readme)中的精髓和难点。

  ![image-20201212232742572](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212232742572.png)

  - 常见的中间件操作

    - 静态服务

      ```js
      app.use(require('koa-static')(__dirname + '/'))
      ```

    - 路由

      ```js
      const router = require('koa-router')()
      router.get('/string', async (ctx, next) => {
        ctx.body = 'koa2 string'
      })
      
      router.get('/json', async (ctx, next) => {
        ctx.body = {
          title: 'koa2 json'
        }
      })
      
      app.use(router.routes())
      ```

    - 日志

      ```js
      app.use(async (ctx,next) => {
        const start = new Date().getTime()
        console.log(`start: ${ctx.url}`);
        await next();
        const end = new Date().getTime()
        console.log(`请求${ctx.url}, 耗时${parseInt(end-start)}ms`)
      }
      ```

### Koa原理

⼀个基于nodejs的入门级http服务，类似下⾯代码：

```js
const http = require('http')

const server = http.createServer((req, res)=>{
  res.writeHead(200)
  res.end('hi')
})

server.listen(3000,()=>{
  console.log('监听端⼝3000')
})
```

![image-20201212233141678](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212233141678.png)

Koa的目标是用更简单化、流程化、模块化的⽅式实现回调部分

```js
// 创建koa.js
const http = require("http");
class Koa {
  listen(...args) {
    const server = http.createServer((req, res) => {
      this.callback(req, res);
    });
    server.listen(...args);
  }
  use(callback) {
    this.callback = callback;
  }
}
module.exports = Koa;

// 调用，index.js
const Koa = require("./koa");
const app = new Koa();
app.use((req, res) => {` `
  res.writeHead(200);
  res.end("hi");
});
app.listen(3000, () => {
  console.log("监听端⼝3000");
});
```

目前为止，Koa只是个马甲，要真正实现目标还需要引入上下文（context）和中间件机制 （middleware）

#### context

Koa为了能够简化API，引入上下文context概念，将原始请求对象req和响应对象res封装并挂载到 context上，并且在context上设置getter和setter，从而简化操作

使用⽅法，接近Koa了

```js
// app.js
app.use(ctx=>{
  ctx.body = 'hehe'
})
```

![image-20201212233812477](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212233812477.png)

- 封装request、response和context

  https://github.com/koajs/koa/blob/master/lib/response.js

```js
// request.js
module.exports = {
  get url() {
    return this.req.url;
  },

  get method(){
    return this.req.method.toLowerCase()
  }
};

// response.js
module.exports = {
  get body() {
    return this._body;
  },
  set body(val) {
    this._body = val;
  }
};

// context.js
module.exports = {
  get url() {
    return this.request.url;
  },
  get body() {
    return this.response.body;
  },
  set body(val) {
    this.response.body = val;
  },
  get method() {
    return this.request.method
  }
};

// koa.js
// 导入这三个类
const context = require("./context");
const request = require("./request");
const response = require("./response");
class Koa {
  listen(...args) {
    const server = http.createServer((req, res) => {
      // 创建上下文
      let ctx = this.createContext(req, res);

      this.callback(ctx)
      // 响应
      res.end(ctx.body);
    });
    // ...
  }
  // 构建上下文, 把res和req都挂载到ctx之上，并且在ctx.req和ctx.request.req同时保存
  createContext(req, res) {
    const ctx = Object.create(context);
    ctx.request = Object.create(request);
    ctx.response = Object.create(response);
    ctx.req = ctx.request.req = req;
    ctx.res = ctx.response.res = res;
    return ctx;
  }
}
```

#### 中间件

Koa中间件机制：Koa中间件机制就是函数式 组合概念 Compose的概念，将⼀组需要顺序执行的函数复合为⼀个函数，外层函数的参数实际是内层函数的返回值。洋葱圈模型可以形象表示这种机制，是[源码](https://github.com/koajs/compose#readme)中的精髓和难点

- 简单函数组合

```js
const add = (x, y) => x + y
const square = z => z * z
const fn = (x, y) => square(add(x, y))
console.log(fn(1, 2))
```

上⾯就算是两次函数组合调用，我们可以把他合并成⼀个函数

```js
const compose = (fn1, fn2) => (...args) => fn2(fn1(...args))
const fn = compose(add,square)
```

- 多个函数组合：中间件的数目是不固定的，我们可以用数组来模拟

```js
const compose = (...[first,...other]) => (...args) => {
  let ret = first(...args)
  other.forEach(fn => {
    ret = fn(ret)
  })
  return ret
}
const fn = compose(add,square)
console.log(fn(1, 2))
```

- 异步中间件：上⾯的函数都是同步的，挨个遍历执行即可，如果是异步的函数呢，是⼀个 promise，我们要⽀持async + await的中间件，所以我们要等异步结束后，再执行下⼀个中间件

```js
function compose(middlewares) {
  return function() {
    return dispatch(0)
    // 执行第0个
    function dispatch(i) {
      let fn = middlewares[i]
      if (!fn) {
        return Promise.resolve()
      }
      return Promise.resolve(
        fn(function next() {
          // promise完成后，再执行下⼀个
          return dispatch(i + 1)
        })
      )
    }
  }
}
async function fn1(next) {
  console.log("fn1")
  await next()
  console.log("end fn1")
}
async function fn2(next) {
  console.log("fn2")
  await delay()
  await next()
  console.log("end fn2")
}
function fn3(next) {
  console.log("fn3")
}
function delay() {
  return new Promise((reslove, reject) => {
    setTimeout(() => {
      reslove()
    }, 2000)
  })
}
const middlewares = [fn1, fn2, fn3]
const finalFn = compose(middlewares)
finalFn()
```

![image-20201212234600299](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212234600299.png)

- compose用在Koa中，koa.js

```js
const http = require("http")
const context = require("./context")
const request = require("./request")
const response = require("./response")

class Koa {
  // 初始化中间件数组
  constructor() {
    this.middlewares = []
  }
  listen(...args) {
    const server = http.createServer(async (req, res) => {
      const ctx = this.createContext(req, res)
      // 中间件合成
      const fn = this.compose(this.middlewares)
      // 执行合成函数并传入上下文
      await fn(ctx)
      res.end(ctx.body)
    })
    server.listen(...args)
  }
  use(middleware) {
    // 将中间件加到数组⾥
    this.middlewares.push(middleware)
  }
  // 合成函数
  compose(middlewares) {
    return function(ctx) { // 传入上下文
      return dispatch(0)
      function dispatch(i) {
        let fn = middlewares[i]
        if (!fn) {
          return Promise.resolve()
        }
        return Promise.resolve(
          fn(ctx, function next() {// 将上下文传入中间件，mid(ctx,next)
            return dispatch(i + 1)
          })
        )
      }
    }
  }
  createContext(req, res) {
    let ctx = Object.create(context)
    ctx.request = Object.create(request)
    ctx.response = Object.create(response)
    ctx.req = ctx.request.req = req
    ctx.res = ctx.response.res = res
    return ctx
  }
}

module.exports = Koa
```

- 使用，app.js

```js
const delay = () => new Promise(resolve => setTimeout(() => resolve() ,2000))
app.use(async (ctx, next) => {
  ctx.body = "1"
  await next()
  ctx.body += "5"
})
app.use(async (ctx, next) => {
  ctx.body += "2"
  await delay()
  await next()
  ctx.body += "4"
})
app.use(async (ctx, next) => {
  ctx.body += "3"
})
```

![image-20201212235026222](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212235026222.png)

#### 常见Koa中间件的实现

- Koa中间件的规范

  - ⼀个async函数
  - 接收ctx和next两个参数
  - 任务结束需要执行next

  ```js
  const mid = async (ctx, next) => {
    // 来到中间件，洋葱圈左边
    next() // 进入其他中间件
    // 再次来到中间件，洋葱圈右边
  }
  ```

- 中间件常见任务：
  - 请求拦截
  - 路由
  - 日志
  - 静态文件服务

##### 路由 router

将来可能的用法

```js
const Koa = require('./koa')
const Router = require('./router')
const app = new Koa()
const router = new Router()

router.get('/index', async ctx => { ctx.body = 'index page' })
router.get('/post', async ctx => { ctx.body = 'post page' })
router.get('/list', async ctx => { ctx.body = 'list page' })
router.post('/index', async ctx => { ctx.body = 'post page' })

// 路由实例输出父中间件 router.routes()
app.use(router.routes())
```

这里是一个策略模式，每一个规则对应一个handler

![image-20201212235738360](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201212235738360.png)

routes()的返回值是⼀个中间件，由于需要用到method，所以需要挂载method到ctx之上，修改request.js

```js
// request.js
module.exports = {
  // add...
  get method(){
    return this.req.method.toLowerCase()
  }
}
```

```js
// context.js
module.exports = {
  // add...
  get method() {
    return this.request.method
  }
}
```

```js
class Router {
  constructor() {
    this.stack = []
  }

  register(path, methods, middleware) {
    let route = {path, methods, middleware}
    this.stack.push(route)
  }
  // 现在只⽀持get和post，其他的同理
  get(path,middleware){
    this.register(path, 'get', middleware)
  }
  post(path,middleware){
    this.register(path, 'post', middleware)
  }
  routes() {
    let stock = this.stack
    return async function(ctx, next) {
      let currentPath = ctx.url
      let route

      for (let i = 0; i < stock.length; i++) {
        let item = stock[i]
        if (currentPath === item.path && item.methods.indexOf(ctx.method) >=
          0) {
          // 判断path和method
          route = item.middleware
          break
        }
      }
      if (typeof route === 'function') {
        route(ctx, next)
        return
      }

      await next()
    }
  }
}
module.exports = Router
```

使用

```js
const Koa = require('./koa')
const Router = require('./router')
const app = new Koa()
const router = new Router()

router.get('/index', async ctx => {
  console.log('index,xx')
  ctx.body = 'index page'
})
router.get('/post', async ctx => { ctx.body = 'post page' })
router.get('/list', async ctx => { ctx.body = 'list page' })
router.post('/index', async ctx => { ctx.body = 'post page' })

// 路由实例输出父中间件 router.routes()
app.use(router.routes())

app.listen(3000,()=>{
  console.log('server runing on port 3000')
})
```

##### 静态文件服务koa-static

- 配置绝对资源目录地址，默认为static
- 获取文件或者目录信息
- 静态文件读取
- 返回

```js
// static.js
const fs = require("fs")
const path = require("path")

module.exports = (dirPath = "./public") => {
  return async (ctx, next) => {
    if (ctx.url.indexOf("/public") === 0) {
      // public开头 读取文件
      const url = path.resolve(__dirname, dirPath)
      const fileBaseName = path.basename(url)
      const filepath = url + ctx.url.replace("/public", "")
      console.log(filepath)
      // console.log(ctx.url,url, filepath, fileBaseName)
      try {
        stats = fs.statSync(filepath)
        if (stats.isDirectory()) {
          const dir = fs.readdirSync(filepath)
          // const
          const ret = ['<div style="padding-left:20px">']
          dir.forEach(filename => {
            console.log(filename)
            // 简单认为不带小数点的格式，就是文件夹，实际应该用statSync
            if (filename.indexOf(".") > -1) {
              ret.push(
                `<p><a style="color:black" href="${
                  ctx.url
                }/${filename}">${filename}</a></p>`
              )
            } else {
              // 文件
              ret.push(
                `<p><a href="${ctx.url}/${filename}">${filename}</a></p>`
              )
            }
          })
          ret.push("</div>")
          ctx.body = ret.join("")
        } else {
          console.log("文件")
          const content = fs.readFileSync(filepath)
          ctx.body = content
        }
      } catch (e) {
        // 报错了 文件不存在
        ctx.body = "404, not found"
      }
    } else {
      // 否则不是静态资源，直接去下⼀个中间件
      await next()
    }
  }
}
```

使用

```js
const static = require('./static')
app.use(static(__dirname + '/public'))
```

##### 请求拦截

```js
// interceptor.js
module.exports = async function(ctx, next) {
  const { res, req } = ctx
  const blackList = ['127.0.0.1']
  const ip = getClientIP(req)

  if (blackList.includes(ip)) { // 出现在⿊名单中将被拒绝
    ctx.body = "not allowed"
  } else {
    await next()
  }
}
function getClientIP(req) {
  return (
    req.headers["x-forwarded-for"] || // 判断是否有反向代理 IP
    req.connection.remoteAddress || // 判断 connection 的远程 IP
    req.socket.remoteAddress || // 判断后端的 socket 的 IP
    req.connection.socket.remoteAddress
  )
}

// app.js
app.use(require("./interceptor"))
app.listen(3000, '0.0.0.0', () => {
  console.log("监听端⼝3000")
})
```

