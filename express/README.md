# Express

相比所有接触过 node 的人都知道 Express 的大名，出自于 TJ 大神之手，它是一个轻量、极简的 Web 框架。我也是一直用它去写一些轻量级的接口，所以一直好奇内部的实现细节，刚好周末有空，所以研究研究。

## 上手

```js
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => res.send('Hello World!'))

app.listen(port, () => console.log(`Example app listening on port ${port}!`))
```

上面代码在本地起了一个 Web 服务，并且监听 3000 端口，同时如果一个 GET 请求打到服务器的话，返回 `Hello World!`。看起来非常简单并且通俗易懂。

## 架构设计

先讲一下 Express 的架构设计吧，贯穿 Express 的核心概念是 Middleware，翻译过来就是中间件的意思，其实就是一个函数，形参的个数以及种类不一样，决定中间件的类型。

1. **应用级别中间件**

```js
var app = express()

app.use(function (req, res, next) {
  console.log('Time:', Date.now())
  next()
})

app.get('/user/:id', function (req, res, next) {
  res.send('USER')
})
```

通过 ` app.use`、`app.METHOD()`（比如 GET, PUT, or POST）注册的中间件可以称之为应用中间件。

2. **路由级别中间件**

```js
var app = express()
var router = express.Router()

router.use(function (req, res, next) {
  console.log('Time:', Date.now())
  next()
})

router.get('/user/:id', function (req, res, next) {
  console.log(req.params.id)
  res.render('special')
})

// 不要忘了挂载的过程！！
app.use('/', router)
```

router 是 Router 实例，它本身也是一个中间件！（后面源码会分析到）。而通过 `router.get`、`router.METHOD()` 等方法绑定的中间件称之为路由中间件。

3. **错误处理中间件**

```js
var app = express()

app.use(function (err, req, res, next) {
  console.error(err.stack)
  res.status(500).send('Something broke!')
})
```

错误处理中间件接收四个参数，必须提供四个形参，因为源码内部就是通过中间件 fn.length 来判断中间件的类型，来决定中间件栈的执行顺序。

4. **内置中间件**

比如 express.static(挂载静态资源)、express.json(处理 application/json 请求)、express.urlencoded(处理类似于表单请求)。

5. **第三发中间件**

还有许多[第三方](http://expressjs.com/en/resources/middleware.html)的中间件。

从上面的中间件来看，对于 Express，它的理念就是万物皆中间件，以及通过 `express()` 返回的 app 也是一个中间件。

**每个中间件都能接触到 req、res，并且还有 next 参数用来将控制权交给下一个中间件，不同的中间件之间可以形成中间件链（middleware chain），中间件链之间还能形成嵌套的关系（比如 express.Router、express()返回的 app）**

举几个例子说一下嵌套的中间件链是什么：

```js
/* 案例一 */
// 将 admin 作为子程序挂载至主程序
var express = require('express')

var app = express() // the main app
var admin = express() // the sub app

admin.get('/', function (req, res) {
  console.log(admin.mountpath) // /admin
  res.send('Admin Homepage')
})
// 挂载！！
app.use('/admin', admin) // mount the sub app

/* 案例二 */
var router = express.Router()

router.use(function (req, res, next) {
  console.log('Time:', Date.now())
  next()
})
router.get('/user/:id', function (req, res, next) {
  res.render('special')
})
// 挂载！！
app.use('/', router)
```

上面两个案例都是调用 `app.use()` 来将中间件挂载到某个路由地址上，而中间件(admin 或 router)上面还保存着自己的中间件，如此，就形成了中间件链嵌套中间件链的关系，类似于树状结构，express 内部会通过一个递归，尽可能的逐步去执行所有的中间件！

理解了上面的这句话，说明你对 express 的架构有了很清晰的概念，剩下要做的就是摸清中间件之间的逻辑流转以及一些细节部分，比如 View、Route、Layer 这些类的作用。

接下来的文章，将以四个维度（Application、Request、Response、Router）尽可能阐述 Express 的技术细节。