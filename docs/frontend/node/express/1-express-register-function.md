---
tags:
  - express
  - nodejs
  - source
  - source-code
---
express 作为nodejs 常用的web framework, 非常便捷及灵活.  本篇就记录一下其实现的一些细节, 来了解其写法变化多样的根本.

查看express的source code, 其实是比较简单的, 可以从创建要给 express的项目, 在 `node_modules` 中的express就有其对应的source code. (当然也免去了从github下载). 

> version of express: 5.1.0


## what express() actually do ?
usually, we start express by this `const app = express()`, so what this function actully do ?


```js
const app = express();

// this function will call: createApplication
// express.js :  export createApplication 
exports = module.exports = createApplication;

// 其实 express() 就是调用此方法
function createApplication() {
 // 可以看到 app 就是要给function, 其函数体就是调用 handle 函数
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };
  // 把 event 函数 .on .emit 函数拷贝到 app
  mixin(app, EventEmitter.prototype, false);
  // 把 application.js 中的 函数拷贝到  app
  mixin(app, proto, false);
  // 定义 request  response 原型函数
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })
	// 调用application.js 中的 init() 函数
  app.init();
  return app;
}
```

由此可以看到, 创建出来的app 其实就是一个function object, 只是为其添加了很多的原型方法.
尤其是`mixin(app, proto, false);`, 此会把平时使用的 `get, post,delete,enable, disable, listen...` 等 所有application.js中的方法复制到 app.  可以简单理解为, app 其实就是application的 门面类, 所以开始时使用到的 express的方法, 大部分都是 application.js 中的方法.

## how the function register into express ?
了解到 app 其实就是 application的一个门面对象, 那么开发时使用的 方法注册, 是如何实现的呢 ?
```js
// register handler for express
app.get('/', (req, res) => {
        res.redirect("/index.html")
        }
    );
```

此真正的实现同样是在 application中, 实现如下:
```js
// MEDTHOS 就是 http中的请发method: get post delete put option etc
var { METHODS } = require('node:http');
exports.methods = METHODS.map((method) => method.toLowerCase());

// 遍历所有的 method, 把其注册为一个 app的属性. 其对应的value 就是 一个 function.
methods.forEach(function (method) {
 // 每个方法(get,post..)对应的都是一个 function, 方法参数为一个 path(即request path)
  app[method] = function (path) {
	 // 如果get 方法只有一个参数, 没有回调 handler, 那么其调用的是 set 方法
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }
    // 把path创建为一个 route 对应
    var route = this.route(path);
    // 设置此 method对应的方法
    // slice.call(arguments, 1) 此会获取到除第一个 path参数外的所有 callback handler
    // route[method].apply(route, ...) 当调用 route[method]时,返回回调函数
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```
通过此方法就可以看到, 每个method(get, post, delete...) 都是app上的一个属性, 
并且`route[method].apply(route, slice.call(arguments, 1));` 会调用到 route中的方法:
```js
// route.js // note: route not router

// 此出的methods同样是 http 请求方法: get post delete option head ...
methods.forEach(function (method) {
 // 每一个 method都注册为了 Route 的要给 属性, 其value为 一个函数, 函数参数为 handler(真正的 reqeust callback function)
  Route.prototype[method] = function (handler) {
   // 获取所有的 回调函数 
    const callbacks = flatten.call(slice.call(arguments), Infinity)
    // 不可以没有回调函数
    if (callbacks.length === 0) {
      throw new TypeError('argument handler is required')
    }
	  // 遍历所有的回调函数, 并把其封装为 wrapper, 并注册到 route中的 stack
    for (let i = 0; i < callbacks.length; i++) {
      const fn = callbacks[i]
      // 如果不是函数对象, 那么 error
      if (typeof fn !== 'function') {
        throw new TypeError('argument handler must be a function')
      }
      debug('%s %s', method, this.path)
      // 封装为 layer, 并把layer注册到 stack中.
      const layer = Layer('/', {}, fn)
      layer.method = method
      this.methods[method] = true
      this.stack.push(layer)
    }
    return this
  }
})
```

到此就可以函数的注册流程:

`app.get(path, handler)` --->  `app[get] function` ---> `Route[get] function`


## how the express find the handler when request come in ?

当请求过来时, 会调用 `app.handle(req,res,next)` 函数, 此就是express的处理开始.

```js
function createApplication() {
  var app = function(req, res, next) {
  // 请求处理开始
    app.handle(req, res, next);
  };
	...
  return app;
}
```

```js
// application.js
app.handle = function handle(req, res, callback) {
  // final handler
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  // set powered by header
  if (this.enabled('x-powered-by')) {
    res.setHeader('X-Powered-By', 'Express');
  }

  // set circular references
  // 记录 请求信息
  req.res = res;
  res.req = req;
  // alter the prototypes
  Object.setPrototypeOf(req, this.request)
  Object.setPrototypeOf(res, this.response)
  // setup locals
  if (!res.locals) {
    res.locals = Object.create(null);
  }
/////  进入router开始处理
  this.router.handle(req, res, done);

};
```

```js
// router.js
Router.prototype.handle = function handle (req, res, callback) {
  debug('dispatching %s %s', req.method, req.url)
  let idx = 0
  let methods
  const protohost = getProtohost(req.url) || ''
	......  // 只保留重点
  // setup basic req values
  req.baseUrl = parentUrl
  req.originalUrl = req.originalUrl || req.url
  next()
  function next (err) {
	.....
    // find next matching layer
    let layer
    let match
    let route
    while (match !== true && idx < stack.length) {
      layer = stack[idx++]
      match = matchLayer(layer, path)
      route = layer.route
		// 找到对应的 route
		....
    }
	...
    // store route for dispatch on change
    if (route) {
      req.route = route
    }
    // Capture one-time layer values
    req.params = self.mergeParams
      ? mergeParams(layer.params, parentParams)
      : layer.params
    const layerPath = layer.path
    // this should be done for the layer
    processParams(self.params, layer, paramcalled, req, res, function (err) {
      if (err) {
        next(layerError || err)
      } else if (route) {
      // 通过封装 route的layer开始继续处理
        layer.handleRequest(req, res, next)
      } else {
        trimPrefix(layer, layerError, layerPath, path)
      }
      sync = 0
    })
  }
```

还记得route是怎么创建的把, 就是在注册 回调函数时, 每一个reqeust path都会被封装为一个Route.
```js
// application.js
methods.forEach(function (method) {
  app[method] = function (path) {
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }
    // path 封装为 route
    var route = this.route(path);
    // 回调函数注册到 Route中
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});

app.route = function route(path) {
// call router的 route函数
  return this.router.route(path);
};

// router.js
Router.prototype.route = function route (path) {
	// 封装 path到 route
  const route = new Route(path)
  // path及route 同样封装为 layer, 并且layer的处理方法为 route.dispatch
  const layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, handle)

// layer 的统一回调函数
  function handle (req, res, next) {
    route.dispatch(req, res, next)
  }
 // 封装route
  layer.route = route
  // 保存layer到 router.stack
  this.stack.push(layer)
  return route
}
```

OK, 回顾到此结束.  那么让router找到 layer后, 调用其`layer.handleRequest` 继续请求的处理:
```js
Layer.prototype.handleRequest = function handleRequest (req, res, next) {
  const fn = this.handle
  if (fn.length > 3) {
    // not a standard request handler
    return next()
  }
  try {
    // invoke function
    // 调用route.dispach 继续进行处理
    const ret = fn(req, res, next)
    // wait for returned promise
    if (isPromise(ret)) {
      if (!(ret instanceof Promise)) {
        deprecate('handlers that are Promise-like are deprecated, use a native Promise instead')
      }
      ret.then(null, function (error) {
        next(error || new Error('Rejected promise'))
      })
    }
  } catch (err) {
    next(err)
  }
}
```


```js
// route.js
Route.prototype.dispatch = function dispatch (req, res, done) {
  let idx = 0
  const stack = this.stack
  let sync = 0
  if (stack.length === 0) {
    return done()
  }
  let method = typeof req.method === 'string'
    ? req.method.toLowerCase()
    : req.method
  if (method === 'head' && !this.methods.head) {
    method = 'get'
  }

  req.route = this
  // 通过next函数查找对应的 layer,
  // 为什么时layer呢？ 因为回调函数同样时封装在 layer中.
  // 所以在route中找到对应的layer, 就可以调用对应的回调函数进行真正的处理
  next()
  function next (err) {
    // signal to exit route
    if (err && err === 'route') {
      return done()
    }
    // signal to exit router
    if (err && err === 'router') {
      return done(err)
    }
    // no more matching layers
    if (idx >= stack.length) {
      return done(err)
    }
    // max sync stack
    if (++sync > 100) {
      return setImmediate(next, err)
    }
    let layer
    let match
    // find next matching layer
    while (match !== true && idx < stack.length) {
    // 查找layer
    /* route注册回调的方法
        // 此处 fn 就是真正的 处理 callback
	    const layer = Layer('/', {}, fn)
	    // method 为 get,post,delete...
       layer.method = method 
    */*
      layer = stack[idx++]
      match = !layer.method || layer.method === method
    }
    // no match
    if (match !== true) {
      return done(err)
    }
    if (err) {
      layer.handleError(err, req, res, next)
    } else {
    // 通过layer 的handleReqest 处理.
      layer.handleRequest(req, res, next)
    }
    sync = 0
  }
}
```

到此就找到了对应的hanler进行request的处理, 处理流程:

![](./images/express_request)



`app.handler(req,res,next)`  ---> `router.handler()` ---> `find layer` ----> `layer.handleRequest(req, res, next)` ---> `route.dispatch(req,res,done)` ---> `find layer` --->  `layer.handleRequest(req, res, next)` ---> `fn(req,res,next)`




