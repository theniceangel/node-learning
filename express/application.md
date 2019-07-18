# Express 源码分析之 Application 篇

Express 模块的到处是一个工厂函数——createApplication，默认会返回 app 函数。

```js
function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };

  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);

  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  app.init();
  return app;
}
```

函数内部逻辑非常简单，内部的两个 mixin，都是为了给 app 这个函数混入一些属性和方法。两个 Object.create 的作用是将 **req** 以及 **res** 模块挂载到 app 的 request 以及 response 属性上。这两个模块在其他的文章会详细分析，现在只用知道 `app.request`、`app.response` 的原型是指向 **req** 以及 **res** 模块。最后调用 app.init()。这个方法定义在 `application.js` 文件。因为执行了第二个 mixin 方法，所以 app 上存在 init 方法。代码如下：

```js
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};

  this.defaultConfiguration();
};
```

声明一些属性为空对象，cache 用来缓存 View 实例，engines 用来存储一些引擎函数 fn。比如 `ejs` 的引擎函数如下：

```js
// 接收 `path`、`options`、`callback` 三个参数
exports.renderFile = function () {
  var args = Array.prototype.slice.call(arguments);
  var filename = args.shift();
  var cb;
  var opts = {filename: filename};
  var data;
  var viewOpts;

  // Do we have a callback?
  if (typeof arguments[arguments.length - 1] == 'function') {
    cb = args.pop();
  }
  if (args.length) {
    data = args.shift();
    if (args.length) {
      utils.shallowCopy(opts, args.pop());
    }
    else {
      if (data.settings) {
        if (data.settings.views) {
          opts.views = data.settings.views;
        }
        if (data.settings['view cache']) {
          opts.cache = true;
        }
        viewOpts = data.settings['view options'];
        if (viewOpts) {
          utils.shallowCopy(opts, viewOpts);
        }
      }
      utils.shallowCopyFromList(opts, data, _OPTS_PASSABLE_WITH_DATA_EXPRESS);
    }
    opts.filename = filename;
  }
  else {
    data = {};
  }

  return tryHandleCache(opts, data, cb);
};
```

tryHandleCache 内部会做模版的编译、渲染工作。

继续关注 `this.defaultConfiguration()` 这段代码的逻辑。

```js
app.defaultConfiguration = function defaultConfiguration() {
  var env = process.env.NODE_ENV || 'development';

  // default settings
  this.enable('x-powered-by');
  this.set('etag', 'weak');
  this.set('env', env);
  this.set('query parser', 'extended');
  this.set('subdomain offset', 2);
  this.set('trust proxy', false);

  // trust proxy inherit back-compat
  Object.defineProperty(this.settings, trustProxyDefaultSymbol, {
    configurable: true,
    value: true
  });

  debug('booting in %s mode', env);

  this.on('mount', function onmount(parent) {
    // inherit trust proxy
    if (this.settings[trustProxyDefaultSymbol] === true
      && typeof parent.settings['trust proxy fn'] === 'function') {
      delete this.settings['trust proxy'];
      delete this.settings['trust proxy fn'];
    }

    // inherit protos
    setPrototypeOf(this.request, parent.request)
    setPrototypeOf(this.response, parent.response)
    setPrototypeOf(this.engines, parent.engines)
    setPrototypeOf(this.settings, parent.settings)
  });

  // setup locals
  this.locals = Object.create(null);

  // top-most app is mounted at /
  this.mountpath = '/';

  // default locals
  this.locals.settings = this.settings;

  // default configuration
  this.set('view', View);
  this.set('views', resolve('views'));
  this.set('jsonp callback name', 'callback');

  if (env === 'production') {
    this.enable('view cache');
  }

  Object.defineProperty(this, 'router', {
    get: function() {
      throw new Error('\'app.router\' is deprecated!\nPlease see the 3.x to 4.x migration guide for details on how to update your app.');
    }
  });
};
```

这段代码主要是做一些初始化的工作，比如设置一些跟 views 有关的配置，如果子 app 挂载到 主 app的时候会触发 'mount' 的回调。在 `app.use` 的逻辑里有这么一段：

```js
fns.forEach(function (fn) {
  // non-express app
  if (!fn || !fn.handle || !fn.set) {
    return router.use(path, fn);
  }
  // ...... 省略
  // fn 就是子 app
  fn.emit('mount', this);
}, this);
```

以上就是 createApplication 的逻辑，对应了我们日常用到的代码。

```js
const express = require('express')
const app = express()
```

但是我们似乎少了一点啥，比如一个 http 服务，如何将 app 与 http 服务关联起来呢，我们来分析下 `app.listen`。

```js
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```

listen 内部会创建一个 http server，并且把 app 作为 http.createServer 的回调函数，并且绑定端口号。

那么完整的例子就如下：

```js
const express = require('express')
const app = express()
app.listen(3000, () => console.log(`Example app listening on port 3000!`))
```
所以这个服务的所有请求都会执行 app 这个函数的逻辑。第一个参数就是 [req](http://nodejs.cn/api/http.html#http_class_http_incomingmessage)，第二个参数就是 [res](http://nodejs.cn/api/http.html#http_class_http_serverresponse)。接下来，真正而又漫长的中间件链执行就要开始了。在这个之前，下两篇文章分别讲一下 `app.request` 的原型对象(req)以及 `app.response` 的原型对象(res)。