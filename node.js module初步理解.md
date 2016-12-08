#node.js module初步理解

在开发一个复杂的应用程序的时候，我们需要把各个功能拆分、封装到不同的文件，在需要的时候引用该文件。没人会写一个几万行代码的文件，这样在可读性、复用性和维护性上都很差，几乎所有的编程语言都有自己的模块组织方式，比如Java中的包、C#中的程序集等，node.js使用模块和包来组织，其机制实现参照了CommonJS标准，虽未完全遵守，但差距不大，使用起来非常简单。

##什么是模块

在node.js中模块与文件是一一对应的，也就是说一个node.js文件就是一个模块，文件内容可能是我们封装好的一些JavaScript方法、JSON数据、编译过的C/C++拓展等,node.js的架构如下：
![](http://images.cnitblog.com/blog/349217/201312/15160642-f5975482ad8641dab0712d26b7118401.png)
其中http、fs、net等都是node.js提供的核心模块，使用C/C++实现，外部用JavaScript封装

##创建、加载模块
模块在node.js中的概念很简单，看看如何创建一个我们自己的模块供开发复用。

在node.js中创建模块非常简单，一个文件就是一个模块，所以我们创建一个test.js文件就创建了一个模块

`test.js`

```js
	var name='';
	
	function setName(n){
	    name=n;
	} 
	
	function printName(){
	    console.log(name);
	}
```

问题是怎么使外部访问这个module，我们知道客户端的JavaScript使用script标签引入JavaScript文件就可以访问其内容了，但这样带了的弊端很多，最大的就是作用域相同，产生冲突问题，以至于前端大师们想出了立即执行函数等方式，利用闭包解决。node.js使用exports和require对象来解决对外提供接口和引用模块的问题。

我们可以把模块中希望被外界访问的内容定义到exports对象中，对test.js稍作修改就可以了

`test.js`

```js
	var name='';
	
	function setName(n){
	    name=n;
	} 
	
	function printName(){
	    console.log(name);
	}
	
	exports.setName=setName;
	exports.printName=printName;
```
这样我们在相同路径下创建index.js，使用require引用一下test.js module

`test.js`

```js
	var test=require('./test');
	
	test.setName('Byron');
	test.printName();
```
![](http://images.cnitblog.com/blog/349217/201312/21163133-34b2c257039a41e7a1edf75285c9121e.png)

###exports一个对象

有时候我们希望模块对外提供的使一个对象，修改一下test.js

`test.js`

```js
	var Student=function(){
	    var name='';
	
	     this.setName=function(n){
	        name=n;
	    }; 
	
	    this.printName=function(){
	        console.log(name)    ;
	    };
	};
	
	exports.Student=Student;
```

这样我们对外提供了一个Student类，在使用的时候需要这样

```js
	var Student=require('./test').Student;
	var student=new Student();
	student.setName('Byron');
	student.printName();
```
require('./test').Student 很丑陋的样子，我们可以简单修改一下exports方式，使require优雅一些

`test.js`

```js
	var Student=function(){
	    var name='';
	
	     this.setName=function(n){
	        name=n;
	    }; 
	
	    this.printName=function(){
	        console.log(name)    ;
	    };
	};
	
	module.exports=Student;
```
这样我们的require语句就可以优雅一些了

```js
	var Student=require('./test');
```

很神奇的样子，不是说好的exports是模块公开的接口嘛，那么module.exports是什么东西？

###module.exports与exports###

事实的情况是酱紫的，其实module.exports才是模块公开的接口，每个模块都会自动创建一个module对象，对象有一个modules的属性，初始值是个空对象{}，module的公开接口就是这个属性——module.exports。既然如此那和exports对象有毛线关系啊！为什么我们也可以通过exports对象来公开接口呢？

为了方便，模块中会有一个exports对象，和module.exports指向同一个变量，所以我们修改exports对象的时候也会修改module.exports对象，这样我们就明白网上盛传的module.exports对象不为空的时候exports对象就自动忽略是怎么回事儿了，因为module.exports通过赋值方式已经和exports对象指向的变量不同了，exports对象怎么改和module.exports对象没关系了。

大概就是这么过程

```js
	module.exports=exports＝{};
	......
	
	module.exports=new Object();
	
	exports=xxx;//和new Object没有关系了，最后返回module.exports，所以改动都无效了

```
###一次加载
无论调用多少次require，对于同一模块node.js只会加载一次，引用多次获取的仍是相同的实例，看个例子
`test.js`

```js
	var name='';
	
	function setName(n){
	    name=n;
	} 
	
	function printName(){
	    console.log(name);
	}
	
	exports.setName=setName;
	exports.printName=printName;

```
`test.js`

```js
	var test1=require('./test'),
	    test2=require('./test');
	
	test1.setName('Byron');
	test2.printName();
```
![](http://images.cnitblog.com/blog/349217/201312/21172710-e3feaf36ddac4394bab966b27f386d48.png)

执行结果并不是''，而是输出了test1设置的名字，虽然引用两次，但是获取的是一个实例

###require搜索module方式
node.js中模块有两种类型：核心模块和文件模块，核心模块直接使用名称获取，比如最长用的http模块

```js
	var http=require('http');
```
在上面例子中我们使用了相对路径 './test'来获取自定义文件模块，那么node.js有几种搜索加载模块方式呢？

1. 核心模块优先级最高，直接使用名字加载，在有命名冲突的时候首先加载核心模块
	文件模块只能按照路径加载（可以省略默认的.js拓展名，不是的话需要显示声明书写）
	1. 绝对路径
	2. 相对路径
2. 查找node_modules目录，我们知道在调用 `npm install <name>` 命令的时候会在当前目录下创建node_module目录(如果不存在) 安装模块，当 require 遇到一个既不是核心模块,又不是以路径形式表示的模块名称时,会试图 在当前目录下的 node_modules 目录中来查找是不是有这样一个模块。如果没有找到,则会 在当前目录的上一层中的 node_modules 目录中继续查找,反复执行这一过程,直到遇到根 目录为止。

--

12.7.16 -
摘录自【[node.js module初步理解](http://www.cnblogs.com/dolphinX/p/3485260.html)】