#Webpack配置杂谈及hash指纹

##1.webpack杂谈部分

**webpack定义**

webpack 是什么？

> *MODULE BUNDLER*

它是这么自称的，***模块打包器***。

在 webpack 里，所有类型的文件都可以是模块，包括我们最常见的 JavaScript，及 CSS 文件、图片、json 文件等等。通过 webpack 的各种加载器，我们可以***更高效地管理这些文件***。

> 用`npm install webpack -g`安装webpack，`webpack --help` 查看帮助文档.如果觉得 npm 安装太慢，可以尝试 npm 的替代工具 [ied](http://gugel.io/ied/) 感觉速度比 npm 快太多了。

**实时刷新**

在 html 文件中引用 bundle.js 文件后，我们有几个问题需要解决。

1. `main.js` 或它所引用的模块的变化如何通知 webpack，重新生成 `bundle.js` ？

    非常简单，在根目录下执行 **webpack --watch** 就可以监控目录下的文件变化并实时重新构建。

2. 上面只是实时构建，我们该如何把结果通知给浏览器页面，让 HTML 页面上的 bundle.js 内容保持最新？

 webpack 提供了 webpack-dev-server 解决实时刷新页面的问题，同时解决实时构建的问题。
 
--
 
webpack-dev-server 提供了[两种模式](http://webpack.github.io/docs/webpack-dev-server.html#automatic-refresh)用于自动刷新页面：
 
 
1. iframe 模式

 我们不访问 http://localhost:8080，而是访问 http://localhost:8080/webpack-dev-server/index.html

2. inline 模式

 在命令行中指定该模式，`webpack-dev-server --inline` 。这样 http://localhost:8080/index.html 页面就会在 js 文件变化后自动刷新了。

以上说的两个页面自动刷新的模式都是指刷新整个页面，相当于点击了浏览器的刷新按钮。

webpack-dev-server 还提供了一种 [--hot 模式](https://webpack.github.io/docs/hot-module-replacement.html)，属于较高阶的应用。

**CSS 加载器**

我们可以按传统方法使用 CSS，即在 HTML 文件中添加：

```js
	<link rel="stylesheet" href="style/app.css">
```

但 webpack 里，CSS 同样可以模块化，使用 `import` 导入。

配置webpack.config.js 文件 

```js
 {
    module: {
        loaders: [
            { test: /\.css$/, loaders: ['style', 'css'] }
        ]
    }
  } 
```

这样，在执行 webpack 后，我们的 CSS 文件就会被打包进 bundle.js 文件中，如果不想它们被打包进去，可以使用 [extract text](https://github.com/webpack/extract-text-webpack-plugin) 扩展。在书写 CSS 的时候，还是需要注意使用[命名规范，比如使用 BEM](https://en.bem.info/methodology/key-concepts/)，否则全局环境 CSS 类的冲突等问题不会消失。

这里，webpack 做了一个[模块化 CSS 的尝试](https://medium.com/seek-developers/the-end-of-global-css-90d2a4a06284#.lmxegaase)，真正意思上的「模块化」，即 CSS 类不会泄露到全局环境中。

##2.hash指纹

	文件的hash指纹通常作为前端静态资源实现增量更新的方案之一，Webpack是目前最流行的开源编译工具之一。
	比如，在Webpack编译输出文件的配置过程中，如果需要为文件加入hash指纹，Webpack提供了两个配置项可供使用：hash和chunkhash。

###2.1 hash与chunkhash


```
 [hash] is replaced by the hash of the compilation.
```
`hash`代表的是compilation的hash值。

```
 [chunkhash] is replaced by the hash of the chunk.
```
`chunkhash`代表的是chunk的hash值

`chunkhash`很好理解，chunk在Webpack中的含义我们都清楚，简单讲，chunk就是模块。`chunkhash`也就是根据模块内容计算出的hash值。

那么该如何理解`hash`是compilation的hash值这句话呢？

####2.1.1 compilation

Webpack官方文档中[How to write a plugin](http://webpack.github.io/docs/how-to-write-a-plugin.html#compiler-and-compilation)章节有对compilation的详解。

compilation对象代表某个版本的资源对应的编译进程。当使用Webpack的development中间件时，每次检测到项目文件有改动就会创建一个compilation，进而能够针对改动生产全新的编译文件。compilation对象包含当前模块资源、待编译文件、有改动的文件和监听依赖的所有信息。

与compilation对应的有个compiler对象，通过对比，可以帮助大家对compilation有更深入的理解。

compiler对象代表的是配置完备的Webpack环境。 compiler对象只在Webpack启动时构建一次，由Webpack组合所有的配置项构建生成。

简单的讲，compiler对象代表的是不变的webpack环境，是针对webpack的；而compilation对象针对的是随时可变的项目文件，只要文件有改动，compilation就会被重新创建。

理解了compilation之后，再回头看`hash`的定义：

```js
[hash] is replaced by the hash of the compilation.

```
compilation在项目中任何一个文件改动后就会被重新创建，然后webpack计算新的compilation的hash值，这个hash值便是`hash`。

如果使用`hash`作为编译输出文件的hash指纹的话，如下：

```js
	output: {
	    filename: '[name].[hash:8].js',
	    path: __dirname + '/built'
	}
```
`hash`是compilation对象计算所得，而不是具体的项目文件计算所得。所以以上配置的编译输出文件，所有的文件名都会使用相同的hash指纹。如下：
![](http://images2015.cnblogs.com/blog/595796/201606/595796-20160628144737468-137488642.png)

这样带来的问题是，三个js文件任何一个改动都会影响另外两个文件的最终文件名。上线后，另外两个文件的浏览器缓存也全部失效。这肯定不是我们想要的结果。

那么如何避免这个问题呢？

答案就是`chunkhash `！

根据`chunkhash `的定义知道，`chunkhash `是根据具体模块文件的内容计算所得的hash值，所以某个文件的改动只会影响它本身的hash指纹，不会影响其他文件。配置webpack的output如下：

```js
	output: {
	    filename: '[name].[chunkhash:8].js',
	    path: __dirname + '/built'
	}
```
编译输出的文件为：
![](http://images2015.cnblogs.com/blog/595796/201606/595796-20160628144750874-61992209.png)

每个文件的hash指纹都不相同，上线后无改动的文件不会失去缓存。

说来说去，好像chunkhash可以完全取代hash，那么hash就毫无用处吗？

###2.1.2 hash应用场景

接上文所述，webpack的`hash`字段是根据每次编译compilation的内容计算所得，也可以理解为项目总体文件的hash值，而不是针对每个具体文件的。

webpack针对compilation提供了两个hash相关的生命周期钩子：`before-hash`和`after-hash`。源码如下：

```js
	this.applyPlugins("before-hash");
	this.createHash();
	this.applyPlugins("after-hash");
```
`hash`可以作为版本控制的一环，将其作为编译输出文件夹的名称统一管理，如下：

```js
	output: {
	    filename: '/dest/[hash]/[name].js'
	}
```

我们不讨论这种方式的合理性和效率，这只是`hash `的一种应用场景。当然，`hash `还有其他的应用场景，不过笔者目前未接触过，欢迎大家补充。


##2.2 js与css共用相同chunkhash的解决方案

webpack的理念是把所有类型的文件都以js为汇聚点，不支持js文件以外的文件为编译入口。所以如果我们要编译style文件，唯一的办法是在js文件中引入style文件。如下：

```js
	import 'style/style.scss';
```

webpack默认将js/style文件统统编译到一个js文件中，可以借助[extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin)将style文件单独编译输出。从这点可以看出，webpack将style文件视为js的一部分。

这样的模式下有个很严重的问题，当我们希望将css单独编译输出并且打上hash指纹，按照前文所述的使用`chunkhash `配置输出文件名时，编译的结果是js和css文件的hash指纹完全相同。不论是单独修改了js代码还是style代码，编译输出的js/css文件都会打上全新的相同的hash指纹。这种状况下我们无法有效的进行版本管理和部署上线。

为什么会产生这种问题呢？

###2.2.1 chunkhash的计算模式

前文提到了webpack的编译理念，webpack将style视为js的一部分，所以在计算chunkhash时，会把所有的js代码和style代码混合在一起计算。比如`main.js`引用了`main.scss`:

```js
	import 'main.scss';
	alert('I am main.js');
```
`main.scss`的内容如下：

```js
	body{
	    color: #000;
	}
```

webpack计算chunkhash时，以main.js文件为编译入口，整个chunk的内容会将main.scss的内容也计算在内：

```js
	body{
	    color: #000;
	}
	alert('I am main.js');
```

所以，不论是修改了js代码还是scss代码，整个chunk的内容都改变了，计算所得的chunkhash自然就不同了。

那么如何解决这种问题呢？

###2.2.2 contenthash

前文提到了使用extract-text-webpack-plugin单独编译输出css文件，造成上一节js/css共用hash指纹的配置为：

```js
	new ExtractTextPlugin('[name].[chunkhash].css');
```

extract-text-webpack-plugin提供了另外一种hash值：`contenthash`。顾名思义，`contenthash `代表的是文本文件内容的hash值，也就是只有style文件的hash值。这个hash值就是解决上述问题的银弹。修改配置如下：

```js
	new ExtractTextPlugin('[name].[contenthash].css');
```

编译输出的js和css文件将会有其独立的hash指纹。

到这里是不是就找到完美的解决方案了呢？

远远没有！

结合上文提到的种种，考虑一下这个问题：**如果只修改了main.scss文件，未修改main.js文件，那么编译输出的js文件的hash指纹会改变吗？**

答案是肯定的。

修改了`main.scss`编译输出的css文件hash指纹理所当然要更新，但是我们并未修改main.js，可是js文件的hash指纹也更新了。这是因为上文提到的：

> webpack计算chunkhash时，以main.js文件为编译入口，整个chunk的内容会将main.scss的内容也计算在内。

那么怎么解决这个问题呢？

很简单，既然我们知道了webpack计算chunkhash的方式，那我们就从这一点出发，尝试修改chunkhash的计算方式。

###2.2.3 chunk-hash

`chunk-hash`并不是webpack中另一种hash值，而是compilation执行生命周期中的一个钩子。chunk-hash钩子代表的是哪个阶段呢？请看[webpack的Compilation.js](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)源码中以下部分：

```js
	for(i = 0; i < chunks.length; i++) {
        chunk = chunks[i];
        var chunkHash = require("crypto").createHash(hashFunction);
        if(outputOptions.hashSalt)
            hash.update(outputOptions.hashSalt);
        chunk.updateHash(chunkHash);
        if(chunk.entry) {
            this.mainTemplate.updateHashForChunk(chunkHash, chunk);
        } else {
            this.chunkTemplate.updateHashForChunk(chunkHash);
        }
        this.applyPlugins("chunk-hash", chunk, chunkHash);
        chunk.hash = chunkHash.digest(hashDigest);
        hash.update(chunk.hash);
        chunk.renderedHash = chunk.hash.substr(0, hashDigestLength);
	}

```

webpack使用NodeJS内置的[crypto](https://nodejs.org/api/crypto.html)模块计算chunkhash，具体使用哪种算法与我们讨论的内容无关，我们只需要关注上述代码中`this.applyPlugins("chunk-hash", chunk, chunkHash);`的执行时机。

chunk-hash是在chunkhash计算完毕之后执行的，这就意味着如果我们在chunk-hash钩子中可以用新的chunkhash替换已存在的值。如下伪代码：

```js
	compilation.plugin("chunk-hash", function(chunk, chunkHash) {
	        var new_hash = md5(chunk);
	        chunkHash.digest = function () {
	        return new_hash;
	    };
	});
```
webpack之所以如果流行的原因之一就是拥有庞大的社区和不计其数的开发者们，实际上，我们遇到的问题已经有先驱者帮我们解决了。插件[webpack-md5-hash](https://github.com/erm0l0v/webpack-md5-hash)便是上述伪代码的具体实现，我们需要做的只是将这个插件加入到webpack的配置中：

```js
	var WebpackMd5Hash = require('webpack-md5-hash');
	
	module.exports = {
	    output: {
	        //...
	        chunkFilename: "[chunkhash].chunk.js"
	    },
	    plugins: [
	        new WebpackMd5Hash()
	    ]
	};
```
***

静态资源的版本管理是前端工程化中非常重要的一环，使用webpack作为构建工具时需要谨慎使用`hash`和`chunkhash `，并且还需要注意webpack将一切视为js模块这种理念带来的一些不便。

webpack可以说是目前最流行的构建工具了，但是其官方文档太过笼统，许多细节并未列出，需要研究源码才会了解。好在我们并非独立战斗，庞大的社区资源也是促进webpack流行的重要因素之一。

行文至此，常规的前端项目中关于静态资源hash指纹的问题基本得到了解决，但是前端的环境是复杂的，各种新技术新框架层出不穷。

--12.8.2016

参考链接: 

1. [Webpack中hash与chunkhash的区别，以及js与css的hash指纹解耦方案](http://www.cnblogs.com/ihardcoder/p/5623411.html)
2. [webpack 教程](https://www.zfanw.com/blog/webpack-tutorial.html)