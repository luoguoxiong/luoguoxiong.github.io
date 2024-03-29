# webpack 插件开发

`webpack` 插件由以下组成：

- 一个 JavaScript 命名函数。
- 在插件函数的 prototype 上定义一个 `apply` 方法。
- 指定一个绑定到 webpack 自身的[事件钩子](https://www.webpackjs.com/api/compiler-hooks/)。
- 处理 webpack 内部实例的特定数据。
- 功能完成后调用 webpack 提供的回调。

## 基本插件结构

```js
function HelloWorldPlugin(options) {
  // 使用 options 设置插件实例……
}

HelloWorldPlugin.prototype.apply = function(compiler) {
  compiler.plugin('done', function() {
    console.log('Hello World!');
  });
};

module.exports = HelloWorldPlugin;
```

### 异步编译插件

```js
function HelloAsyncPlugin(options) {}

HelloAsyncPlugin.prototype.apply = function(compiler) {
  compiler.plugin("emit", function(compilation, callback) {

    // 做一些异步处理……
    setTimeout(function() {
      console.log("Done with async work...");
      callback();
    }, 1000);

  });
};

module.exports = HelloAsyncPlugin;
```

### 检索遍历资源(asset)、chunk、模块和依赖(实现依赖分析)

```js
unction MyPlugin() {}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    // 检索每个（构建输出的）chunk：
    compilation.chunks.forEach(function(chunk) {
      // 检索 chunk 中（内置输入的）的每个模块：
      chunk.modules.forEach(function(module) {
        // 检索模块中包含的每个源文件路径：
        module.fileDependencies.forEach(function(filepath) {
          // 我们现在已经对源结构有不少了解……
        });
      });

      // 检索由 chunk 生成的每个资源(asset)文件名：
      chunk.files.forEach(function(filename) {
        // Get the asset source for each file generated by the chunk:
        var source = compilation.assets[filename].source();
      });
    });

    callback();
  });
};

module.exports = MyPlugin;
```

### 监听观察图(watch graph)的修改

```js
function MyPlugin() {
  this.startTime = Date.now();
  this.prevTimestamps = {};
}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    var changedFiles = Object.keys(compilation.fileTimestamps).filter(function(watchfile) {
      return (this.prevTimestamps[watchfile] || this.startTime) < (compilation.fileTimestamps[watchfile] || Infinity);
    }.bind(this));

    this.prevTimestamps = compilation.fileTimestamps;
    callback();
  }.bind(this));
};

module.exports = MyPlugin;
```

### 监听chunk修改

```js
function MyPlugin() {
  this.chunkVersions = {};
}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    var changedChunks = compilation.chunks.filter(function(chunk) {
      var oldVersion = this.chunkVersions[chunk.name];
      this.chunkVersions[chunk.name] = chunk.hash;
      return chunk.hash !== oldVersion;
    }.bind(this));

    callback();
  }.bind(this));
};

module.exports = MyPlugin;
```

### 总结

在插件开发需要对webpack 的 compiler、compilation 有基本的认识

`compiler` 对象代表了完整的 webpack 环境配置。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。

`compilation` 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用。

https://www.webpackjs.com/api/compiler-hooks/

https://www.webpackjs.com/api/compilation-hooks/

https://www.webpackjs.com/api/plugins/