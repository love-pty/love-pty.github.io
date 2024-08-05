---
description: JavaScript是一门无类（class-free）语言，也正因为如此，生搬单例模式的概念并无意义。在JavaScript中创建对象的方法非常简单，既然我们只需要一个“唯一”的对象，为什么要为它先创建一个“类”呢？这无异于穿棉衣洗澡，传统的单例模式实现在JavaScript中并不适用。单例模式的核心是确保只有一个实例，并提供全局访问。
---
# 详解JavaScript中的单例模式

## 什么是单例模式？
单例模式的定义是：保证一个类仅有一个实例，并提供一个访问它的全局访问点。单例模式是一种常用的模式，有一些对象我们往往只需要一个，比如线程池、全局缓存、浏览器中的window对象等。在JavaScript开发中，单例模式的用途同样非常广泛。试想一下，当我们单击登录按钮的时候，页面中会出现一个登录浮窗，而这个登录浮窗是唯一的，无论单击多少次登录按钮，这个浮窗都只会被创建一次，那么这个登录浮窗就适合用单例模式来创建。

## 如何实现一个单例模式
要实现一个单例模式非常简单，无非使用一个变量来标志是否为某个类创造过一个实现，如果没有，就创造一个，如果有，就直接返回之前创造的那个。

```javascript
function Singleton() {
    this.instance = null
}

Singleton.prototype.getInstance = function() {
    return this.instance || (this.instance = new Singleton())
}

var s1 = Singleton.getInstance()
```
通过以上代码我们实现了一个简单的单例模式，通过getInstance方法来获取单例对象，这似乎与以往获取对象和创建对象的方式有所区别，如果用以往通过`new Singleton()`的方式来创建对象，那我们得到的对象还符合单例模式吗？
答案是不符合的，这段代码具有一定的“不透明性”他要求使用者在使用这段代码的时候必须通过getInstance方法来获取对象。常规的`new xxx()`并不可行。
## 透明的单例模式
一个完美的单例模式是“透明的”，也就是说，使用者在使用的时候不需要知道单例模式的存在。
们现在的目标是实现一个“透明”的单例类，用户从这个类中创建对象的时候，可以像使用其他任何普通类一样。在下面的例子中，我们将使用CreateDiv单例类，它的作用是负责在页面中创建唯一的div节点。
```javascript
var CreateDiv = (function() {
    var instance = null
    var CreateDiv = function() {
        if (instance) {
            return instance
        }
        this.create()
        return instance = this
    }
    CreateDiv.prototype.create = function() {
        var div = document.createElement('div')
        div.innerHTML = 'hello world'
        document.body.appendChild(div)
    }
    return CreateDiv
})()
```
虽然现在完成了一个透明的单例类的编写，但它同样有一些缺点。为了把instance封装起来，我们使用了自执行的匿名函数和闭包，并且让这个匿名函数返回真正的Singleton构造方法，这增加了一些程序的复杂度，阅读起来也不是很舒服。

为了把instance封装起来，我们使用了自执行的匿名函数和闭包，并且让这个匿名函数返回真正的Singleton构造方法，这增加了一些程序的复杂度，阅读起来也不是很舒服。

然现在完成了一个透明的单例类的编写，但它同样有一些缺点。为了把instance封装起来，我们使用了自执行的匿名函数和闭包，并且让这个匿名函数返回真正的Singleton构造方法，这增加了一些程序的复杂度，阅读起来也不是很舒服。

## 用代理实现单例模式
实际上单例模式的构造函数做了两件事情，第一件事情是创建一个对象，第二件事情是确保返回对象的唯一性。
这是每个单例模式都会做的事情，那我们就可以将这个步骤抽离出来。
我们依然使用上述代码，把管理单例模式的代码删除，使其成为一个普通的创建dom节点的类。
```javascript
function CreateDiv() {
    this.create()
}
CreateDiv.prototype.create = function() {
    var div = document.createElement('div')
    div.innerHTML = 'hello world'
    document.body.appendChild(div)
}
```
接下来引入代理类proxySingletonCreateDiv：
```javascript
var proxySingletonCreateDiv = (function() {
    var instance = null
    return function(Constructor,...args) {
        return instance || (instance = new Constructor(...args))
    }
})()
var a = new proxySingletonCreateDiv(CreateDiv)
```
## JavaScript中的单例模式
JavaScipt是一门面向对象的语言，但是JavaScript中的面向对象却不是基于类的，JavaScript中的对象是基于原型的。这意味着JavaScript在创建一个对象时并没有实例化一个类，而是基于原型继承。因此基于类的单例模式在JavaScript中并不适用。JavaScript是一门无类（class-free）语言，也正因为如此，生搬单例模式的概念并无意义。在JavaScript中创建对象的方法非常简单，既然我们只需要一个“唯一”的对象，为什么要为它先创建一个“类”呢？这无异于穿棉衣洗澡，传统的单例模式实现在JavaScript中并不适用。
**单例模式的核心是确保只有一个实例，并提供全局访问。**

## 惰性单例
惰性单例指的是在需要的时候才创建对象实例。惰性单例是单例模式的重点，这种技术在实际开发中非常有用，有用的程度可能超出了我们的想象。
设我们是WebQQ的开发人员​，当点击导航里QQ头像时，会弹出一个登录浮窗​，很明显这个浮窗在页面里总是唯一的，不可能出现同时存在两个登录窗口的情况。
第一种解决方案是在页面加载完成的时候便创建好这个div浮窗，这个浮窗一开始肯定是隐藏状态的，当用户点击登录按钮的时候，它才开始显示：
```html
<html>
    <body>
      <button id="loginBtn">登录</button>
    </body>
    <script>
        var loginLayer = (function(){
          var div = document.createElement( 'div' )
          div.innerHTML = ’我是登录浮窗’
          div.style.display = 'none'
          document.body.appendChild( div )
          return div;
        })()
        document.getElementById( 'loginBtn' ).onclick = function(){
          loginLayer.style.display = 'block'
        };
    </script>
</html>
```
这种方式有一个问题，也许我们进入WebQQ只是玩玩游戏或者看看天气，根本不需要进行登录操作，因为登录浮窗总是一开始就被创建好，那么很有可能将白白浪费一些DOM节点。
现在改写一下代码，使用户点击登录按钮的时候才开始创建该浮窗：
```javascript
var createLoginLayer = (function(){
    var div;
    return function(){
      if ( ! div ){
          div = document.createElement( 'div' )
          div.innerHTML = ’我是登录浮窗’
          div.style.display = 'none'
          document.body.appendChild( div )
      }
          return div
    }
})()
document.getElementById( 'loginBtn' ).onclick = function(){
    var loginLayer = createLoginLayer()
    loginLayer.style.display = 'block'
}
```
## 通用的惰性单例
上一节我们完成了一个可用的惰性单例，但是我们发现它还有如下一些问题。
* 这段代码仍然是违反单一职责原则的，创建对象和管理单例的逻辑都放在createLoginLayer对象内部。
* 如果我们下次需要创建页面中唯一的i​frame，或者script标签，用来跨域请求数据，就必须得如法炮制，把createLoginLayer函数几乎照抄一遍。

我们需要把不变的部分隔离出来，先不考虑创建一个div和创建一个i​frame有多少差异，管理单例的逻辑其实是完全可以抽象出来的，这个逻辑始终是一样的：用一个变量来标志是否创建过对象，如果是，则在下次直接返回这个已经创建好的对象：
```javascript
var obj
if ( ! obj ){
  obj = xxx
}
```
现在我们就把如何管理单例的逻辑从原来的代码中抽离出来，这些逻辑被封装在getSingle函数内部，创建对象的方法fn被当成参数动态传入getSingle函数：
```javascript
var getSingle = function( fn ){
    var result
    return function(){
      return result || ( result = fn .apply(this, arguments ) )
    }
}
```
接下来将用于创建登录浮窗的方法用参数fn的形式传入getSingle，我们不仅可以传入createLoginLayer，还能传入createScript、createIframe、createXhr等。之后再让getSingle返回一个新的函数，并且用一个变量resul​t来保存fn的计算结果。resul​t变量因为身在闭包中，它永远不会被销毁。在将来的请求中，如果resul​t已经被赋值，那么它将返回这个值。代码如下：
```javascript
var createLoginLayer = function(){
    var div = document.createElement( 'div' )
    div.innerHTML = ’我是登录浮窗’
    div.style.display = 'none'
    document.body.appendChild( div )
    return div
}
var createSingleLoginLayer = getSingle( createLoginLayer )
document.getElementById( 'loginBtn' ).onclick = function() {
    var loginLayer = createSingleLoginLayer()
    loginLayer.style.display = 'block'
}
```

> 参考：曾探《JavaScript设计模式与开发实践》