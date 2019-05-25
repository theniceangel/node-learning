# graceful-js

由于内置的 nodejs 的 `fs` 原生模块在不同环境与平台的的表现有一些差异，为了抹平这些差异，`graceful-js` 就诞生了，它解决了如下的问题：

- `open` 与 `readdir` 方法被队列化了，因为这些方法可能导致系统打开太多的文件描述符，而抛出 [EMFILE](https://nodejs.org/dist/latest-v10.x/docs/api/errors.html) 错误。内部会在某一个文件被关闭的时候，会重新调用这些方法。
- 修复 `node v0.6.2` 之前版本的 `lchmod` 的问题。
- 实现 `fs.lutimes` 方法。
- 如果用户不是 `root` 权限，在调用 `chown`、`fchown`、`lchown` 直接忽略 `EINVAL`、`EPERM` 错误。
- 调用 `read` 遇到 `EAGAIN` 时候会重试。

## 使用

```js
// use just like fs
var fs = require('graceful-fs')

// now go and do stuff with it...
fs.readFileSync('some-file-or-whatever')
```

## patch

```js
var realFs = require('fs')
var gracefulFs = require('graceful-fs')
gracefulFs.gracefulify(realFs)
```

## 源码分析

源码的实现如下：

- **debug**

  ```js
  function noop () {}

  var debug = noop
  if (util.debuglog)
    debug = util.debuglog('gfs4')
  else if (/\bgfs4\b/i.test(process.env.NODE_DEBUG || ''))
    debug = function() {
      var m = util.format.apply(util, arguments)
      m = 'GFS4: ' + m.split(/\n/).join('\nGFS4: ')
      console.error(m)
    }

  if (/\bgfs4\b/i.test(process.env.NODE_DEBUG || '')) {
    process.on('exit', function() {
      debug(queue)
      require('assert').equal(queue.length, 0)
    })
  }
  ```

  这一步是定义 `debug` 函数。默认是调用 node 的 `util.debuglog` 内置函数，这个函数是要求在启动 node 的时候就得设置好环境变量 `process.env.NODE_DEBUG`。比如 `NODE_DEBUG=gfs4 node test.js`。而 `else if` 的逻辑判断是为了兼容在调用 `debug` 之前就设置 `process.env.NODE_DEBUG = 'gfs4'` 环境变量的语法。

- **模块的导出**

  ```js
  module.exports = patch(clone(fs))
  if (process.env.TEST_GRACEFUL_FS_GLOBAL_PATCH && !fs.__patched) {
      module.exports = patch(fs)
      fs.__patched = true;
  }
  ```

  这一步是默认导出 fs 的 clone 版本并且再由 patch 函数处理之后的 fs。所以每次调用 `graceful.js` 导出的函数是会返回不同的 fs 对象。

  再看 `patch` 函数。

  ```js
  function patch (fs) {

    polyfills(fs)
    fs.gracefulify = patch
    fs.FileReadStream = ReadStream;  // Legacy name.
    fs.FileWriteStream = WriteStream;  // Legacy name.
    fs.createReadStream = createReadStream
    fs.createWriteStream = createWriteStream
    var fs$readFile = fs.readFile
    fs.readFile = readFile
    function readFile (path, options, cb) {
      if (typeof options === 'function')
        cb = options, options = null

      return go$readFile(path, options, cb)

      function go$readFile (path, options, cb) {
        return fs$readFile(path, options, function (err) {
          if (err && (err.code === 'EMFILE' || err.code === 'ENFILE'))
            enqueue([go$readFile, [path, options, cb]])
          else {
            if (typeof cb === 'function')
              cb.apply(this, arguments)
            retry()
          }
        })
      }
    }

    var fs$writeFile = fs.writeFile
    fs.writeFile = writeFile
    function writeFile (path, data, options, cb) {
      if (typeof options === 'function')
        cb = options, options = null

      return go$writeFile(path, data, options, cb)

      function go$writeFile (path, data, options, cb) {
        return fs$writeFile(path, data, options, function (err) {
          if (err && (err.code === 'EMFILE' || err.code === 'ENFILE'))
            enqueue([go$writeFile, [path, data, options, cb]])
          else {
            if (typeof cb === 'function')
              cb.apply(this, arguments)
            retry()
          }
        })
      }
    }

    var fs$appendFile = fs.appendFile
    if (fs$appendFile)
      fs.appendFile = appendFile
    function appendFile (path, data, options, cb) {
      if (typeof options === 'function')
        cb = options, options = null

      return go$appendFile(path, data, options, cb)

      function go$appendFile (path, data, options, cb) {
        return fs$appendFile(path, data, options, function (err) {
          if (err && (err.code === 'EMFILE' || err.code === 'ENFILE'))
            enqueue([go$appendFile, [path, data, options, cb]])
          else {
            if (typeof cb === 'function')
              cb.apply(this, arguments)
            retry()
          }
        })
      }
    }

    var fs$readdir = fs.readdir
    fs.readdir = readdir
    function readdir (path, options, cb) {
      var args = [path]
      if (typeof options !== 'function') {
        args.push(options)
      } else {
        cb = options
      }
      args.push(go$readdir$cb)

      return go$readdir(args)

      function go$readdir$cb (err, files) {
        if (files && files.sort)
          files.sort()

        if (err && (err.code === 'EMFILE' || err.code === 'ENFILE'))
          enqueue([go$readdir, [args]])

        else {
          if (typeof cb === 'function')
            cb.apply(this, arguments)
          retry()
        }
      }
    }

    function go$readdir (args) {
      return fs$readdir.apply(fs, args)
    }

    if (process.version.substr(0, 4) === 'v0.8') {
      var legStreams = legacy(fs)
      ReadStream = legStreams.ReadStream
      WriteStream = legStreams.WriteStream
    }

    var fs$ReadStream = fs.ReadStream
    if (fs$ReadStream) {
      ReadStream.prototype = Object.create(fs$ReadStream.prototype)
      ReadStream.prototype.open = ReadStream$open
    }

    var fs$WriteStream = fs.WriteStream
    if (fs$WriteStream) {
      WriteStream.prototype = Object.create(fs$WriteStream.prototype)
      WriteStream.prototype.open = WriteStream$open
    }

    fs.ReadStream = ReadStream
    fs.WriteStream = WriteStream

    function ReadStream (path, options) {
      if (this instanceof ReadStream)
        return fs$ReadStream.apply(this, arguments), this
      else
        return ReadStream.apply(Object.create(ReadStream.prototype), arguments)
    }

    function ReadStream$open () {
      var that = this
      open(that.path, that.flags, that.mode, function (err, fd) {
        if (err) {
          if (that.autoClose)
            that.destroy()

          that.emit('error', err)
        } else {
          that.fd = fd
          that.emit('open', fd)
          that.read()
        }
      })
    }

    function WriteStream (path, options) {
      if (this instanceof WriteStream)
        return fs$WriteStream.apply(this, arguments), this
      else
        return WriteStream.apply(Object.create(WriteStream.prototype), arguments)
    }

    function WriteStream$open () {
      var that = this
      open(that.path, that.flags, that.mode, function (err, fd) {
        if (err) {
          that.destroy()
          that.emit('error', err)
        } else {
          that.fd = fd
          that.emit('open', fd)
        }
      })
    }

    function createReadStream (path, options) {
      return new ReadStream(path, options)
    }

    function createWriteStream (path, options) {
      return new WriteStream(path, options)
    }

    var fs$open = fs.open
    fs.open = open
    function open (path, flags, mode, cb) {
      if (typeof mode === 'function')
        cb = mode, mode = null

      return go$open(path, flags, mode, cb)

      function go$open (path, flags, mode, cb) {
        return fs$open(path, flags, mode, function (err, fd) {
          if (err && (err.code === 'EMFILE' || err.code === 'ENFILE'))
            enqueue([go$open, [path, flags, mode, cb]])
          else {
            if (typeof cb === 'function')
              cb.apply(this, arguments)
            retry()
          }
        })
      }
    }

    return fs
  }
  ```

  1. 首先调用 `polyfills` 函数，将 fs 对象传进去，来看看 `polyfills` 函数的实现。

      ```js
      function patch (fs) {
      // (re-)implement some things that are known busted or missing.

      // lchmod, broken prior to 0.6.2
      // back-port the fix here.
      if (constants.hasOwnProperty('O_SYMLINK') &&
          process.version.match(/^v0\.6\.[0-2]|^v0\.5\./)) {
        patchLchmod(fs)
      }

      // lutimes implementation, or no-op
      if (!fs.lutimes) {
        patchLutimes(fs)
      }

      // https://github.com/isaacs/node-graceful-fs/issues/4
      // Chown should not fail on einval or eperm if non-root.
      // It should not fail on enosys ever, as this just indicates
      // that a fs doesn't support the intended operation.

      fs.chown = chownFix(fs.chown)
      fs.fchown = chownFix(fs.fchown)
      fs.lchown = chownFix(fs.lchown)

      fs.chmod = chmodFix(fs.chmod)
      fs.fchmod = chmodFix(fs.fchmod)
      fs.lchmod = chmodFix(fs.lchmod)

      fs.chownSync = chownFixSync(fs.chownSync)
      fs.fchownSync = chownFixSync(fs.fchownSync)
      fs.lchownSync = chownFixSync(fs.lchownSync)

      fs.chmodSync = chmodFixSync(fs.chmodSync)
      fs.fchmodSync = chmodFixSync(fs.fchmodSync)
      fs.lchmodSync = chmodFixSync(fs.lchmodSync)

      fs.stat = statFix(fs.stat)
      fs.fstat = statFix(fs.fstat)
      fs.lstat = statFix(fs.lstat)

      fs.statSync = statFixSync(fs.statSync)
      fs.fstatSync = statFixSync(fs.fstatSync)
      fs.lstatSync = statFixSync(fs.lstatSync)

      // if lchmod/lchown do not exist, then make them no-ops
      if (!fs.lchmod) {
        fs.lchmod = function (path, mode, cb) {
          if (cb) process.nextTick(cb)
        }
        fs.lchmodSync = function () {}
      }
      if (!fs.lchown) {
        fs.lchown = function (path, uid, gid, cb) {
          if (cb) process.nextTick(cb)
        }
        fs.lchownSync = function () {}
      }
      // ... 省略其他部分
      ```

      从 `polyfills` 来看，就是兼容一些特殊版本的 node 和跨平台下的一些 API 兼容问题，让 `fs` 使用起来更优雅。

  2. 再向外暴露一个 `gracefulify` 方法，用来 patch `fs` 或者 类似于 `fs` 结构的对象。**但是这种做法会导致原生的 `fs` 或者 你传入的 `fs-like` 的库被改写**。
  3. 之后部分就是对文件系统的部分 API 的 patch了。

而且导出的还有其他的 API。比如：

```js
// Always patch fs.close/closeSync, because we want to
// retry() whenever a close happens *anywhere* in the program.
// This is essential when multiple graceful-fs instances are
// in play at the same time.
module.exports.close = (function (fs$close) { return function (fd, cb) {
  return fs$close.call(fs, fd, function (err) {
    if (!err)
      retry()

    if (typeof cb === 'function')
      cb.apply(this, arguments)
  })
}})(fs.close)

module.exports.closeSync = (function (fs$closeSync) { return function (fd) {
  // Note that graceful-fs also retries when fs.closeSync() fails.
  // Looks like a bug to me, although it's probably a harmless one.
  var rval = fs$closeSync.apply(fs, arguments)
  retry()
  return rval
}})(fs.closeSync)
```

`close` 与 `closeSync` 都是用的原生 `fs` 模块的方法，因为 `graceful-js` 被引用多次之后，会生成不同的 fs 对象，但是期望在程序的任何地方都能够在调用这两个方法的时候执行 `retry` 的逻辑。这也就是开头讲到的问题

> `open` 与 `readdir` 方法被队列化了，因为这些方法可能导致系统打开太多的文件描述符，而抛出 [EMFILE](https://nodejs.org/dist/latest-v10.x/docs/api/errors.html) 错误。内部会在某一个文件被关闭的时候，会重新调用这些方法。

所以任何由 `graceful-js` 生成的 `fs` 对象，都应该保持同一个 `close` 与 `closeSync` 的引用，这样就能在某个文件被关闭的时候，重新的打开调之前用 `open`、`readdir` 方法的文件。