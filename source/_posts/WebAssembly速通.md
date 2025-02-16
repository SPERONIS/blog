---
title: WebAssembly速通
date: 2024-03-05 12:08:10
tags: 视野分享
---


## 前言
最近接了一个业务，要将一个端上的能力搞到微信小程序上，在调研后决定使用 WebAssembly 方式接入，很快就把这块的技术调研和技术方案设计完成了，但相关的技术文章较少，本文分享下 WebAssembly 相关的基础知识和一些在项目中使用的踩坑经验。

## WebAssembly
引用官网的定义: WebAssembly(缩写为 Wasm) 是一种基于堆栈的虚拟机的二进制指令格式。 Wasm 被设计为编程语言的可移植编译目标，支持在网络上部署客户端和服务器应用程序。目前我们所接触到的 WebAssembly 更多的是 WebAssembly 及其生态;与传统的 JavaScript 不同，WebAssembly 可以提供更高的性能，特别是在需要执行计算密集型任务的情况下。它可以将C 和 C++ 代码打包成可执行的字节码，然后在 Web 浏览器中运行，以提供更快的加载速度和更高的性能。此外，WebAssembly 还支持模块化编程，使开发人员能够更轻松地管理代码，并且具有更高的可移植性。



[WebAssembly MDN](https://developer.mozilla.org/en-US/docs/WebAssembly)



## 生态语言
从 WebAssembly 问世以来就展现出巨大的发展潜力，受到业界的广泛关注，各大语言也是积极探索对 WebAssembly 的支持，从老大哥 C/C++、javaScript 到新兴的 GO、Rust 等语言，各种编辑工具和生态链层出不穷，一副百花齐放的态势，通过社区的发展，WebAssembly 制定并发布了 WebAssembly System Interface 标准(简称 WASI)后，WebAssembly 的运行环境甚至可以获取到 io 等能力，可见不远的未来服务端也将成为下个WebAssembly 的战场。





## 生态工具链
在了解工具链前先认识下 LLVM(Low Level Virtual Machine) 它是一个可重用的编译器和工具链基础设施。它最初是作为编译器前端和优化器的框架而设计的，现在已经发展成为一个完整的编译器基础设施，支持多种编程语言和多个平台。LLVM 的设计目标是提供一个高度灵活的编译器基础设施，具有可重用性、可扩展性和性能优势。采用了模块化的架构将编译过程分为多个阶段，并使用中间表示(intermediate Representation，IR)来表示代码,这种设计使得 LLVM 可以在不同的编译器阶段进行优化、分析和转换，并且可以支持多种目标平台和编程语言如C/C++、Swit、Rust 等，通过浏览器把 wasm 文件加载并将字节码继续编译为浏览器所运行的设备的机,器码使其运行。


我们常见的工具有: Emscripten(C/C++)，Wasm-pack(Rust)等，限于篇幅本文就只介绍最常用的Emscripten:
Emscripten 基于 LLVM libcxx 实现了一套能提供大部分 C/C++ 标准库的能力，此外还支持多线程、库导入等方式将大部分 C/C++ 工程无缝迁移到 WebAssembly 上。





## Hello World
老规矩，我们从 0 开始，以一个 helo worid 来进入 WebAssembly 世界(这里使用 C/C++ 语言，Emscripten 工具，MacOS 进行阐述，其他工具和环境原理类似)

1. 前置工具，安装打工人套件: Git、CMake、Python 等
2. 安装Emscripten: [https://emscripten,org/docs/getting_started/downloads.html](https://emscripten,org/docs/getting_started/downloads.html);

```bash
# Get the emsdk repo
git clone https://github.com/emscripten-core/emsdk.git
# Enter that directory
cd emsdk
# Fetch the latest version of the emsdk (not needed the first time you clone)9 
git pull
# Download and install the latest SDk tools.
./emsdk install latest
# Make the "latest" SDK "active" for the current user. (writes .emscripten file)
./emsdk activate latest
# Activate PATH and other environment variables in the current terminal
source ./emsdk env.sh
```



3. 创建 hello.c 文件，示例代码:

```c
#include <stdio.h>
int main(){
  printf("Hello World\n");
  return 0;
}

```



4. 执行编译

```bash
 emcc hello.c -o hello.html
```

编译后将生成一个 .wasm 文件，一个 js 胶水代码文件提供一个全局 Module 对象，封装了

WebAssembly 的一系列操作方法; 以及一个 .htm l模板文件



### 进一步配置
接下来我们继续探索 Emscripten 强大的能力吧:

导出 C++ 方法给到 js 层调用

```c
#include <stdio.h>
#include <emscripten/emscripten.h> // 导入头文件

int main() {
    printf("Hello World\n"); 
}
return 0;
// 定义方法导出
#ifdef _cplusplus
#define EXTERN  extern "c"
#else
#define EXTERN
#endif

// 导出方法
EXTERN EMSCRIPTEN KEEPALIVE int sayHello(string argc, char ** argv) {
    printf("sayHello called\n");
    return 0;
}

```

执行编译后可以看到在.wasm文件中通过export 导出的方法，在js 胶水代码中也能看到私有方法

```c
var _sayHello = Module['_sayHello'] = createExportWrapper('sayHello');
```

既然是私有方法，肯定不希望外部直接调用，所以我们可以在编译时指定导出具体的 API:

```c
 emcc -o hello,html hello.C -S NO EXIT RUNTIME=1 -S "EXPORTED RUNTIME METHODS=['ccall']"
```

在js中可以通过Module,ccall 调用该 API:

```javascript
const result = Module.ccall(
  "sayHello", // name of C function
  null, // return type
  null, // argument types
  null, // arguments
);
console.log('result',result); // result 0
```

同时，我们也可以指定 HTML 模板，在模板代码中使用 `{{{ SCRIPT}}}` 匹配插入 `<script async type="text/javascript"src="hello.js"></script>` 可以结合前端语言进行页面展示。

```html
<!doctype html>
<html lang="en-us">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>Emscripten-Generated Code</title>
  </head>
  <body>
    <!-- {{{ SCRIPT }}} -->
    <button id="mybutton">点击输入</button>
    <script>
      document.getElementById("mybutton").addEventListener("click",()=> {
        const result = Module.ccall(
          "sayHello", // name of c function
          ['sting'],  // return type
          ['sting'],  // argument types
          ['Tom'], // arguments
        ):
        console.log('result',result); // 0
      });
    </script>
  </body>
</html>
```

执行编译命令:

```bash
 emcc -o hello.html hello.c --shell-file demo/template.html -S NO EXIT RUNTIME=1 -S "EXPORTED RUNTIME METHODS=['ccall']"
```


感受到 WebAssembly 的魅力了吗，接下来我们再看几个强大的 Emscripten 编译指令:

1. WASM

```bash
# WASM=0 生成 asm.js 格式(适用于 WebAssembly 不支持的情况)
# WASM=1 生成包含 wasm 格式
# WASM=2 asm.js 与 wasm 格式均生成，添加支持判定，优先使用 wasm 格式。
-S WASM=1
```

2. ENVIRONMENT

```bash
 # 设定当前的运行环境，避免生成的 js 文件中判定环境，并运行不同的代码
 # web - web 浏览器运行环境。
 #worker - web worker 支持的运行环境,
 # node - Node 环境.
 # shell -js 脚本运行环境，如 d8、jsc
 # 如果为空则支持所有运行时环境
 -S ENVIRONMENT='web'; 
```

3.内存模式

```bash
# 指定内存的大小  
-S INITIAL MEMORY=134217728
# 限制内存大小
-S MAXIMUM MEMORY=134217728
# 是否内存会增长
-S ALLOW MEMORY GROWTH=0/1
```

4. MODULARIZE

```bash
# 导出模块名称，通过常和 MODULARIZE_INSTANCE / MODULARIZE 配合使用
-S EXPORT NAME='Module':
# 是否生成模块 instance 单粒，返回格式为:{}
-S MODULARIZE INSTANCE=0
# 是否生成模块，返回 function 格式
-S MODULARIZE=0;
```

5. EXPORT ES6

```bash
# 使用 ES6 模块导出而不是 UMD 导出进行导出。 必须为 ES6 导出启用 MODULARIZE。需要注意低版本浏览器可能不支持它。
-S EXPORT ES6=1
```

6. preload-file

```bash
# 该命令可以以 preload 模式打包文件或者文件夹，通常可以指定较大的依赖文件或者文本(如.js、.bin 等)
# 执行 --preload-file 后将生成 .data 文件，C/C++ 代码可以通过 fopen 或 xhr 的方式加载并读取文件内容并执行，相比 --embed-file 效率高得多
--preload-file hello.text

```

Emscripten 本文只列出我们在生产中几个常用的编译指令，更多功能可自行参阅官网说明。

## 项目实战
最简单的方式就是在 HTML 或者 Reactue 等工程中使用，如通过以下指令进行编译

```bash
mCC-S WASM=1 -S WASM_BIGINT -S ENVIRONMENT=Web -S MODULARIZE=1 -S EXPORT_ES6=1 -S USE_ES6_IMOPRT_META=1 -s EXPORET_NAME="loadWASM" -s EXPORT_RUNTIME_METHODS="['ccal']" -s EXPORT_FUNTIONS="['_malloc', '_free']" --shell-file demo/template.html
``` 

在 hellojs 中我们可以看到 loadWASM 以具名方式导出:

```javaScript
var loadWASM = (() => {
  var scriptDir = import.meta.url;

  return (
    function(moduleArg={}){

      //  ..
  )
})();

export default loadWASM;

```

在 hello.html 则可以使用 ES6 方式导入

```html
<!DOCTYPE html>
<html lang="en">
  <head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title></title>
  <script type="module">
    import initModule from"./video index.js";
    initModule(Module);
  </script>
  </head>

  <body>
  </body>
</html>
```

更推荐使用以下方式，通过 Promise 保证在 js 加载完成后得到可用的 Module 对象，同时，WebAssembly 也提供了onRuntimeInitialized 生命周期来保证胶水代码初始化完成

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title></title>
<script type="module">

  import loadWASM from"./video index.js";

  loadWASM().then(module => {

  window.Module = modute;

  });
</script>
</head>
<body>
</body>
</html>
```

js 如何将参数传给 C/C++，一般情况下 js 向 C/C++ 传值可直接根据类型传即可:

```js
// demo.js
const result = Module.ccall(
  "getResult",   // name of c function
  null,         // return type
  ['number'],  // argument types
  [111]，      // arguments
);
```

但在某些场景下，我们需要将json 字符串、二进制等大量数据传给 C/C++，这时可以使用 WebAssembly 提供的 malloc 申请内存方式，将参数放在指针的内存中传递给 C/C++:

```c
hello.c
# include <stdio.h>
# include <emscripten/emscripten.h>
// int Main...
void getResult(const char* ptr){
  printf("%s"，ptr)
}
```

```javaScript
// demo.js
// 序列化数据，字符串在C/C++ 中是一个以 0 结尾的字符数组
const jsonstr = Module.intArrayFromString(JSON,stringify(data)).concat(0);
// 分配内存空间
const ptr = Module. malloc(jsonstr.length);
// 将数据存入新内存中
Module.HEAPU8.set(jsonstr, ptr);
const result = Module.ccall(
  "getResult", // name of c function
  null, // return type
  ['number'], // argument types
  [ptr]，// arguments
);
// 一定要记住释放
Module._free(ptr);
```

除了 js 将数据传给 C/C++，反过来也有 C/C++ 向 js 传 字符串等数据

```javaScript
// demo.js
// 需要提前知道内存大小，可以使用 C/C++ 提供直接获取某个参数内存大小的方式，也可以通过约定的方式
const size = Module.ccall('getSize',['number'], null, null);
const ptr=Module. malloc(size);

const result = Module.ccall(
  "getResult", // name of C function
  null,// return type
  ['number'], // argument types
  [ptr]，// arguments
);
// 取值
const resBuffer = Module.HEAPU8.subarray(ptr, ptr + size);
const str = new TextDecoder().decode(resBuffer);
console.log(str);
// 一定要记住释放16 Module. free(ptr);

```

一定要记住一个原则: 谁分配内存，就由谁释放内存!!! 不管什么程序语言，内存生命周期基本是一致的: 分配你所需要的内存、使用分配到的内存(读、写)、不需要时将其释放\归还。在生产环境中，内存泄露的问题较难排除，需要在开发时进行严格要求。



### 深入
--preload-file ，在生产环境中，建议使用预加载的方式将一些依赖模块或者文件前面提到了一个指令:进行管理，使用的 js 胶水层会通过 xhr 的方式进行装载，既能加快加载效率，也能减少打包体积:

```javaScript
function fetchRemotePackage (packageName, packageSize, callback, errBack) {
  var xhr = new XMLHttpRequest();
  xhr.open('GET', packageName, true);
  xhr.responseType ='arraybuffer';
  xhr.onprogress =function(event) {
    var url = packageName;
    var size = packageSize;
    if (event.total) {
      size = event.total;
    }
    if (event.loaded) {
      if (!xhr.addedTotal) {
        xhr.addedTotal = true;
        if (!Module['dataFileDownloads']) {
          Module['dataFileDownloads'] = {};
        }
        Module.dataFileDownloads[url] = {
          total: size,
          loaded: 0,
        };
      } else {
        Module.dataFileDownloads[url].loaded = event.loaded;
      }
      var total = 0;
      var loaded = 0;
      var num = 0;
      for (var download in Module.dataFileDownloads) {
        var data =Module.dataFileDownloads[download];
        total += data.total;
        loaded += data.loaded;
        num++;
      }
      total = Math.ceil(total * Module.expectedDataFileDownloads / num);
      if (Module['setStatus']) {
        Module['setStatus'](`Downloading data... (${loader}/${total})`); 
      }
    } else if (!Module.dataFileDownloads) {
      if (Module['setStatus']){
        Module['setStatus']('Downloading data...');
      }
    };
    xhr.onerror = function(event) {
      throw new Error("NetworkError for:" + packageName):
    }
    xhr.onload = function(event) {
      if (xhr.status ==200 || xhr,status ==304 || xhr.status == 206 || (xhr.status === 0 && xhr.response)) {
        packageData=xhr.response;
        callback(packageData);
      } else {
        throw new Error(xhr.statusText+":"+ xhr.responseURL);
      }
    }
    xhr.send(null);
  }

function handleError(error) {
  console.error(error);
  throw error;
}

var fetchedCallback = null;
var fetched = Module['getPreloadedPackage'] ? Module['getPreloadedPackage'](REMOTE_PACKAGE_NAME, REMOTE_PACKAGE_SIZE) : null; 

if (!fetched) {
  fetchRemotePackage(REMOTE_PACKAGE_NAME, REMOTE_PACKAGE_SIZE, function(data) {
    if (fetchedCallback) {
      fetchedCallback(data);
      fetchedCallback = null;
    } esle {
      fetched = data;
    }
  }, handleError);
}

```

在工程中使用时记得将其构建为静态资源文件

```js
// webpack.config.js
const CopyPlugin = require('copy-webpack-plugin');
module,exports =()=>({
  plugins: [
    new CopyPlugin({
      patterns: [
        {
          from: path.resolve(__dirname, 'src/wasm/hello.data'),
          to: path.resolve(__dirname, 'dist/hello.data'),
        },
      ],
    })
  ]

```

如果你是通过 npm 包等形式发布 bundle 文件，可以通过修改胶水代码中的 `REMOTE_PACKAGE_BASE` 参数替换 `.data` 文件域名（比如发布到 cdn）来加速静态资源的下载。

## 在小程序中使用

除了在浏览器容器中使用 WebAssembly，各小程序厂商如微信、支付宝等都已支持在各自的小程序中使用WebAssembly。但是在各小程序中进行了不同程度的封装，接下来看看如何在微信小程序中将 WebAssembly引

首先看下微信小程序官方的异同描述:

WXWebAssembly.instantiate(path, imports)

和标准 WebAssembly.instantiate 类似，差别是第一个参数只接受一个字符串类型的代码包路径，指向代码包内 .wasm 文件

与 WebAssembly 的异同

1、WxWebAssembly.instantiate(path,imports)方法，path为代码包内路径(支持.wasm和.wasm.br后缀)

2、支持 WxWebAssembly.Memory

3.支持 WxWebAssembly.Table

4、支持WXWebAssembly.Global

5、expont 支持函数、Memory、Table,iOS 平台暂不支持 Global

但是，要想在微信小程序中运行 WebAssembly，只看上面的描述远不能简单实现接入，必须要对 js 胶水层的进行一定的改造来完成接入

1. 在微信小程序中 `import.meta.url` 获取不到，可以直接改为 "/"，注意 `.wasm` 文件在微信小程序中的文件路径即可:
```js
// hello.js
wasmBinaryFile = new URL('hello.wasm', import.meta.url).href;
wasmBinaryFile ='hello.wasm';
```

2. 微信小程序中提供的是 WXWebAssembly，可以定义一个全局变量

```js
 const isWechat = typeof wx== 'object';
 const WebAssembly =isWechat ? WXWebAssembly : WebAssembly;
```

3. 微信小程序不支持 xhr，需使用 `wx.request` 代替:
``` js
function fetchRemotePackage(packageName, packageSize, callback, errBack) {
  if (isWechat) {
    wx.request({
    url: packageName,
    responseType: 'arraybuffer',
    success: function(packageData){
      callback(packageData);
    });
  }
}
```

4. WXWebAssembly.instantiate(path,imports) 处理：
```js
function instantiateArrayBuffer(binaryFile, imports, receiver, info) {
  var r;
  if (isWechat) {
    r=WebAssembly.instantiate('/'+ wasmBinaryFile, info);
  } else {
    r= getBinaryPromise().then(function(binary){
      return WebAssembly.instantiate(binary, info);
    }).then(function(instance){
      return instance;
    });
  }

  return r.then(receiver, (reason) => {
    err(`failed to asynchronously prepare wasm: ${reason}`);

    // Warn on some common problems.
    if (isFileURI(wasmBinaryFile)) {
      err(`warning: Loading from a file URl (${wasmBinaryFile}) is not supported in mos`);
    }
    abort(reason);
  });
}
```

5. 其他，如:

  a. `performance.now` 不支持，没有特殊需求可以使用 `Date.now` 替代

  b. Document、TextDecoder、TextEncoder 等不支持，可以写代码 hack 掉或者使用第三方库替代。在复杂的前端工程中引入 WebAssembly 可能会涉及到更多的工作，需要充分评估需求以及现有的代码，并确定哪些部分可以从引入 WebAssembly 中受益; 当然，必须注意代码的兼容性和性能测试，以保证引入 WebAssembly 不会引入新的问题。

## 最后

在需要执行高性能计算和数据密集型任务的场景中 WebAssembly 可以帮助我们实现更快的视频解码和编码;也可以帮助我们实现更快的游戏加载和渲染;甚至在安全防控方面也可以帮助我们实现更高效的安全检测和攻击防御以及未来结合大模型等在端上进行机器学习。

