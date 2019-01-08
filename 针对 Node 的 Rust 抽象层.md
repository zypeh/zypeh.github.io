---
title: 针对 Node 的 Rust 抽象层
date: 2016-01-26
---

> Node.js 是谷歌 V8 引擎、libuv平台抽象层 以及主体使用 Javscript 编写的核心库三者集合的一个包装外壳。” 除此之外，值得注意的是，Node.js 的作者瑞恩·达尔 (Ryan Dahl) 的目标是创建具有实时推送能力的网站。在 Node.js 中，他给了开发者一个使用事件驱动来实现异步开发的优秀解决方案。

Javscript 在 IO 密集的应用场景例如 RESTful 环境下，Node.js 能发挥高并发，足以应付巨量的请求。但是你绝对不会用它来做 CPU 密集的工作吧？

所以主流的是用 C++, C 来为 Node.js 编写原生模块 (native modules)。但是鉴于现在个人学习新秀语言 Rust ，因此打算说下两者之间如何互相使用。

## 谈谈 C++ 语言和 V8 引擎的关系

```javascript
module.exports.hello = function() { return 'hello from C++'; };
```

这样的 Javascript 代码用 C++ 利用 v8 标准库下。

```cpp
// hello.cc
#include <node.h>

using namespace v8;

void Method(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = Isolate::GetCurrent();
  HandleScope scope(isolate);
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world"));
}

void init(Handle<Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);
}

NODE_MODULE(addon, init)
```

根据源码， `method` 接受 callbacks 然后在内部进行 Handle 控制，大体就是把 args 的 `return value` 设成 JS 里的物件类型, 在这里我们设它为 _world_。

`NODE_MODULE` 就是注册一个 Node Module, addon 就是其文件名。之后会编译成 _addon.node_ (以 `.node` 做后缀)

## Rust 语言简单介绍

Rust 语言是一门强调安全、并发、高效的系统编程语言。个人认为是个值得期待的系统编程语言之一。因为其对底层内存控制和安全把捏的很好，这里我们会用它来编写我们的 node 模块。因为编写个 Nodejs 模块最主要的原因还是因为 javascript 不能很高效地完成内存和大量运算，安全才是 Rust 语言的卖点。

## Neon

Neon 是近期很新颖的一个 Node 抽象层。从语法的角度来说简化了许多，而且还针对 Node.js 的环境，Neon 保证其 JS 堆和 Rust 的栈能完美的被垃圾回收机制跟踪。

    考虑效率问题，因为 JS FFI (Foreign Function Interface) 开销也是很大的，尽可能用原生 JS, 万不得已才考虑。
    代码为：
    ```js
    function() { return Math.floor(133.7/PI); }
    ```

安装
```bash
$ npm install -g neon-cli
```
创建新的项目
```bash
$ neon-cli new <project_name>
```

## 探讨 neon 所搭建的项目

其中有 package.json, README.md, 值得记下的有这几个

路径             | 内容
:---------------:|:------------:
`./lib`          | 里面含有 JS 代码
`./native`       | 内含 Rust 语言的包 (包括 Cargo.toml, /src 等等)
`./node_modules` | 里面包含所有 node modules, 你的需求都在这

我们看看 `lib` 的代码

```js
var addon = require('../native');
console.log(addon.hello());
```

没啥特别的，就简单的从 native 路径加载模块。

我们看看 `native` 里的源码

```rust
#[macro_use]
extern crate neon;

use neon::vm::{Call, JsResult, Module};
use neon::js::JsString;

fn hello(call: Call) -> JsResult<JsString> {
  let scope = call.scope;
  Ok(JsString::new(scope, "hello node").unwrap())
}

register_module!(m, {
  m.export("hello", hello)
});
```

这就是简单的打印 hello node 的 nodejs 模块。

`register_module!` 是个宏 (macro)，用来注册 module 用。`Call` 就是一个 Rust 的物件，封装了 JS 和 Rust 的内存。Neon 函数就会接受 `Call` 然后返回 `JsResult` 或是 `throw` 的异常值。

#### 参考资料
[Nodejs](http://wiki.jikexueyuan.com/project/nodejs/addons.html)
[Javascript 里有个 C](https://segmentfault.com/a/1190000000453606)
[node-cpp-modules // github](https://github.com/kkaefer/node-cpp-modules) - 这里有许多的 comment 和例子
[neon // github ](https://github.com/rustbridge/neon)
