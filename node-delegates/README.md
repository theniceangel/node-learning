# node-delegates

node-delegates 库出自于 [TJ](https://github.com/tj/node-delegates) 大神之手，知道这个库是在学习 koa 源码的时候看到的，这个库主要是为了解决嵌套属性提供类似于 shorthand 的方法，为了使得代码更简洁。举个例子：

```js
var delegate = require('delegates');
let proto = {
  request: {
    getHeaders () {
      return 'headers'
    }
    get _headersSent () {
      return true
    }
  }
}

delegate(proto, 'request')
  .method('getHeaders') // 代理 getHeaders 方法至 proto
  .getter('_headersSent') // 代理 _headersSent 属性至 proto

proto.getHeaders() // 输出 'headers'
proto._headersSent // 输出 true
```

从上面的例子可以看出，`proto.request` 都以 shorthand 的方式代理到 `proto`，这样使用起来更简洁。

为了吃透内部的实现逻辑，所以翻了一下源码，从 API 的维度来分析一波。

## Delegate(proto, prop)

```js
function Delegator(proto, target) {
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  this.proto = proto;
  this.target = target;
  this.methods = [];
  this.getters = [];
  this.setters = [];
  this.fluents = [];
}
```

返回一个 delegator 实例。 `this.proto` 是要代理的对象，`this.target` 是代理对象的根属性。

## Delegate#method(name)

```js
Delegator.prototype.method = function(name){
  var proto = this.proto;
  var target = this.target;
  this.methods.push(name);

  proto[name] = function(){
    return this[target][name].apply(this[target], arguments);
  };

  return this;
};
```

method 方法的作用正如它的中文含义一样，代理嵌套的方法至 proto。比如 `proto.getHeaders()` 就是 `proto.request.getHeaders()` 的别名。

## Delegate#getter(name) & #setter(name)

```js
// getter
Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);

  proto.__defineGetter__(name, function(){
    return this[target][name];
  });

  return this;
};

// setter
Delegator.prototype.setter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);

  proto.__defineSetter__(name, function(val){
    return this[target][name] = val;
  });

  return this;
};
```

代理 \_\_defineSetter__ 和 \_\_defineGetter__，拿上面的例子来说，就是 `proto._headersSent` 求值的过程也就是对 `proto.request._headersSent` 进行求值。

## Delegate#access(name)

```js
Delegator.prototype.access = function(name){
  return this.getter(name).setter(name);
};
```

`delegator.getter` 与 `delegator.setter` 的功能合集。

## Delegate#fluent(name)

```js
Delegator.prototype.fluent = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.fluents.push(name);

  proto[name] = function(val){
    if ('undefined' != typeof val) {
      this[target][name] = val;
      return this;
    } else {
      return this[target][name];
    }
  };

  return this;
};
```

类似于 jquery 的 getter 和 setter 功能，如果调用方法不传参，那么就是 getter，否则就是 setter。举个例子

```js
let a = 1
let proto = {
  request: {
    query: 'zhang'
  }
}
delegate(proto, 'request')
  .fluent('query')

proto.query() // 输出 'jz'

proto.query('li') // proto.request.query为 'li'
```

## Delegate.auto(proto, targetProto, targetProp)

```js
Delegator.auto = function(proto, targetProto, targetProp){
  var delegator = Delegator(proto, targetProp);
  var properties = Object.getOwnPropertyNames(targetProto);
  for (var i = 0; i < properties.length; i++) {
    var property = properties[i];
    var descriptor = Object.getOwnPropertyDescriptor(targetProto, property);
    if (descriptor.get) {
      delegator.getter(property);
    }
    if (descriptor.set) {
      delegator.setter(property);
    }
    if (descriptor.hasOwnProperty('value')) { // could be undefined but writable
      var value = descriptor.value;
      if (value instanceof Function) {
        delegator.method(property);
      } else {
        delegator.getter(property);
      }
      if (descriptor.writable) {
        delegator.setter(property);
      }
    }
  }
};
```

auto 方法又是对 method、setter、getter 的更高层次的代理，它的功能就是相当于这些的合集。
