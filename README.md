Nirvana v3
====================
移动端Webapp开发框架

#### usage
首先你需要创建一个HTML文档，然后在head中插入：
```html
<style>
* {
    margin: 0;
    padding: 0;
    list-style: none;
    font-weight: normal;
    font-family: normal;
    box-sizing: border-box;
    -webkit-tap-highlight-color: transparent;
}
.view {
    display: none;
}
.view.current {
    display: block;
}
</style>
```
这个基础样式是控制视图的显示和隐藏。

然后创建一个 **app.js** 引入到你的页面中：
```html
<script src="app.js"></script>
```
在 **app.js** 中引入我们的nirvana.js：
```js
var app = require('nirvana');
```
在继续之前，你需要了解下面几个概念：

#### View
首先我们需要了解整个的移动Webapp是由一个个的内容 **视图** 组成，通过用户输入的网址或链接跳转进行不同内容的展示实际上就是不同 **视图** 的切换。

那么我们至少需要创建一个视图：
```js
var app = require('nirvana');
app.view('home', {
    action: function(req, res){
        this.node.innerHTML = '我是Home!';
        console.log('视图被激活');
    }
});
```
这样我们就有了一个可显示的画面。
相关列表: `app[':views']`

#### Router
当我们有了这个视图后，如何显示在页面上呢？
这就需要我们另一个概念， **路由** 负责根据用户访问的网址，提供不同的内容视图给用户浏览。

那么我们现在就需要这样一个路由：
```js
var app = require('nirvana');
app.view('home', {
    action: function(req, res){
        this.node.innerHTML = '我是Home!';
        console.log('视图被激活');
    }
});
app.use('home', app.view('home'));
app.listen();
```
这个代码会将 **#home** 绑定访问home视图。
路由的定义由 **/** 间隔，如：
`list/movie`
也可以用变量表示，如：
`list/:movie`
可以匹配 `#list/123` 或者 `#list/test`
那么`View`的`action`接收到的`req.params.movie` 就会相应的等于`123`或`test`，具体的`req`属性请输出`view`的`req`了解。

app.listen(); 表示开始进行地址侦听并启动app渲染。

相关列表: `app[':routes']`
#### Error
当用户访问的地址无法匹配到路由或者出现其他错误的时候，会将问题抛送给Error对象，你也可以自定义自己的错误。
```js
var app = require('nirvana');
app.view('home', {
    action: function(req, res){
        this.node.innerHTML = '我是Home!';
        console.log('视图被激活');
    }
});
app.use('home', app.view('home'));
app.error('404', function(req){
    console.log('无法找到:"' + req.path + '"');
    app.redirect('home');
});
app.listen();
```
已经预定义的错误：
`*:` 任意错误都会调用的...
`404:` 无法找到匹配的路由；


定义和抛出一个新错误：
```js
//定义一个错误"500"(可以是任意字符标识)
app.error('500', function(obj){
    console.log('服务器错误...');
});

app.error('*', function(obj){
    console.log('任意错误都会抛出...');
});

//抛出错误
app.error('500');// app.error('500', {错误对象...});
```
Error错误定义结合AOP机制，为同一个错误标识定义多个处理函数。
相关列表: `app[':error']`

#### Template
如果需要在View里显示比较复杂的HTML内容，则需要使用到前端模板功能。
现在很多前端模板其实就是把HTML内容预编译成Javascript的模板函数。
把一个模板函数注册到app中:
```js
//引入nirvana
var app = require('nirvana');
//定义模板
app.tmpl('layout', function(obj){
    return '模板函数返回:' + obj.name;
});
//定义视图
app.view('home', {
    action: function(req, res){
        //使用模板
        this.node.innerHTML = app.tmpl('layout', {name: 'home'});
        console.log('视图被激活');
    }
});
//定义路由
app.use('home', app.view('home'));
//定义错误
app.error('404', function(req){
    app.redirect('home');
});
//开始侦听
app.listen();
```
这里我直接写了一个函数, 你可以直接引用其他模板引擎的模板函数。
例如FIS:
`app.tmpl('layout', __inline('tmpl/layout.tmpl'));`

#### Directive
我们使用模板可以展示不同的内容，但是一旦我们模板中的HTML节点需要绑定事件就会比较麻烦，**指令集** 可以帮我们解决这个问题。

注册一个指令:
```js
app.directive('nv-alert', function(e){
    alert(this.getAttribute(e.directive.name));
});
```
**指令集** 会为事件对象增加一个`directive`的属性，用来保存当前触发的指令信息。
具体参数可以`console.log`查看。
当你注册完指令后，就可以在模板的HTML节点中增加相应指令的属性：
```html
<a href="javascript:void(0);" nv-alert="这是一个指令">调用指令</a>
```
这样当用户触发这个链接的`click`事件，同事会触发`nv-alert`指令。

当我们想修改指令的事件信息时，需要稍微复杂些的指令设置:
```js
app.directive('nv-alert', {
    context: document.body,
    type: 'click',
    action: function(e){
        alert(this.getAttribute(e.directive.name));
    }
});
```
`context:` 是指事件委托给上级DOM哪个目标节点去侦听(可省略，默认值为body)；
`type:` 是在触发指令的事件名, 可用空格间隔输入多个(可省略, 默认值为click)；
`action:` 触发指令时执行的函数；

#### Cache
通常我们的Webapp都需要从另外的接口获取数据, 我们内置了一个简单的jsonp函数就可以做到，但是在一定的周期内，我们期望只拉取一次数据缓存到本地。

建立一个数据缓存:
```js
//一个简单的数据缓存
app.cache('data', 'http://...');
//更高级的缓存设置
app.cache('data', {
    params: {...},
    remote: 'http://...',
    maxAge: 180,
    storage: window.sessionStorage,
    action: emptyFunction
});
```

`remote:` 数据来源接口，Webapp会发出JSONP请求到这个地址获取数据；
`params:` 获取远程数据时携带的参数，同一缓存中不同参数会在本地存储多份缓存；
`maxAge:` 本地缓存数据的生命周期，可省略，默认为3分钟（单位：秒）；
`storage:` 本地缓存接口对象，可省略，默认sessionStorage；
`action:` 取回远程数据后的预处理函数，可省略；

你可以通过`action`来对远程获取回来的新数据进行裁剪和转换，最后`action`函数的返回值才会进行缓存和应用。

如何使用缓存的数据:
```js
//无需修改参数的读取
app.cache('data', function(){
    //读取数据成功
}, function(){
    //读取数据失败
});

//同一缓存源，参数不同的数据读取
app.cache('data', {
    //参数
}, function(){
    //读取数据成功
}, function(){
    //读取数据失败
});
```
获取数据失败只有在本地没有可用缓存，并且网络请求失败的情况下。
另外，缓存的声明周期是以本地时间的三分钟后的实际时间为准，若修改本地时间到较早的日期则无法更新缓存。







