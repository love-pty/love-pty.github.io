---
description: 发布订阅模式是一种在软件设计中广泛使用的消息传递模式。
---
# 详解发布-订阅模式

## 什么是发布-订阅模式

发布订阅模式是一种在软件设计中广泛使用的消息传递模式。发布订阅模式定义了一种一对多的依赖关系，允许多个订阅者对象同时监听某一个发布者或主题对象。当发布者（或主题对象）的状态发生变化或产生新的消息时，它会将这些变化或消息通知给所有订阅了该发布者或主题的订阅者对象，从而使它们能够自动更新自己的状态或执行相应的操作。

发布-订阅模式中涉及到三个角色：

- **发布者（Publisher）**：发布消息或事件的一方，它负责将消息或事件发送到中介（如消息队列、事件总线）中。发布者不需要知道有哪些订阅者会接收这些消息。
- **订阅者（Subscriber）**：接收并处理消息或事件的一方，它向中介注册自己感兴趣的消息类型或主题，并在接收到相应的消息时执行相应的操作。订阅者也不需要知道消息是由哪个发布者发布的。
- **中介（Broker/Topic/Event Bus）**：作为发布者和订阅者之间的桥梁，负责存储和转发消息。中介可以是消息队列、事件总线等，它根据消息的类型或主题将消息分发给相应的订阅者。

## 例子：订阅咖啡店新品通知

巴拉巴拉一大堆，抽象的语言总是难以理解，下面我举个例子来解释发布-订阅模式

假设你居住在一个充满创意咖啡店的社区，你特别喜欢尝试新口味的咖啡，但你又不想每天去每家咖啡店询问是否有新品上市。幸运的是，这个社区有一个集中的“咖啡新品信息中心”（即中介），负责收集和分发各咖啡店的新品信息。

发布者

- 在这个例子中，各家咖啡店就是发布者。每当他们研发出新口味的咖啡时，他们就会将新品信息（包括咖啡名称、口味描述、上架时间等）发送给“咖啡新品信息中心”。

订阅者

- 你就是订阅者之一。你向“咖啡新品信息中心”注册了你的联系方式（如邮箱或手机号）和你感兴趣的咖啡店列表。这样，每当这些咖啡店有新品上市时，你就会收到通知。

中介

- “咖啡新品信息中心”在这个例子中充当了中介的角色。它负责接收来自各家咖啡店的新品信息，并根据订阅者的兴趣进行筛选和分发。它可能是一个实体机构，也可能是一个在线平台或应用程序。

### 整体流程：

- 顾客使用“咖啡新品信息中心”提供的平台或机构对一些咖啡店进行订阅。
- 当这些咖啡店研发出新口味时通过“咖啡新品信息中心”通知给顾客。

在这里有一个关键：顾客只有在订阅之后才能收到新品上线的消息，顾客永远看不到订阅之前发布的信息。这意味着：发布-订阅的流程应该是**先订阅再发布**。

## 如何实现

在上述例子中”联系方式“成为接收信息的工具，在程序设计中我们使用函数来进行信息的接收。

顾客订阅时传递一个函数，当咖啡店想要发布时调用这个函数。

比如这样：

```javascript
class EventBus{
    events = {}
    on(event,callback) {	//event事件名称（咖啡店名称），callback信息传递工具（联系方式）
        this.events[event] = callback
    }
    emit(event,...args) {
        this.events[event](...args)
    }
}
const eventBus = new EventBus()
eventBus.on('A店最新咖啡'，(info)=>{
    console.log(info)
    //新品咖啡上线，好喝不贵
})
eventBus.emit('A店最新咖啡','新品咖啡上线，好喝不贵')
```

好啦，现在家里的座机已经可以成功的订阅咖啡店并且收到最新的发布信息。但当你出门在外是并不能接收到最新的发布信息，这让你感到烦恼。下面让我们改进一下将手机号码也加入：

上一段代码中`events`是普通对象，这并不方便。

改进`events`的数据结构，`events = new Map()`将event事件名称（咖啡店名称）作为键。

由于我们需要使用多个联系方式，所以`Map`中的值我们放入一个由`callback`组成的`Set`对象。

```javascript
class EventBus {
    
    constructor() {
        this.events = new Map();
    }
    
    emit(event,...args) {
        this.events.has(event) && this.events.get(event).forEach(callback => callback(...args));
    }

    on(event,callback) {
        this.events.has(event) ? this.events.get(event).add(callback) : this.events.set(event,new Set([callback]));
    }
}
const eventBus = new EventBus()
eventBus.on('A店最新咖啡'，(info)=>{
    console.log(info)
    //新品咖啡上线，好喝不贵
})
eventBus.on('A店最新咖啡'，(info)=>{
    console.log(info)
    //新品咖啡上线，好喝不贵
})
eventBus.emit('A店最新咖啡','新品咖啡上线，好喝不贵')
```

这样，家里的座机和随身携带的手机都能接收到咖啡店发布的最新消息了。

某一天，咖啡店老板淘气的小儿子将同一款咖啡的信息发布了N次，这不仅让小儿子的屁股遭了罪，还让你的收件箱变成了99+。接下来要做了是，将相同咖啡的消息接收设置为仅一次。

接着改进：

```javascript
class EventBus {
    
    constructor() {
        this.events = new Map();
    }
    
    emit(event,...args) {
        this.events.has(event) && this.events.get(event).forEach(callback => callback(...args));
    }

    on(event,callback) {
        this.events.has(event) ? this.events.get(event).add(callback) : this.events.set(event,new Set([callback]));
    }
    
    off(event,callback) {
        this.events.has(event) && this.events.get(event).delete(callback);
    }

    once(event,callback) {	//在callback外部包装一层函数，callback执行完成之后删除此函数，这样函数将不会被存储在events中。
        const fn = (...args) => {
            callback(...args)
            this.off(event,fn)
        }
        this.on(event,fn)
    }
    
}
const eventBus = new EventBus()
eventBus.once('A店最新咖啡',(info) => {
    console.log(info)
    //新品咖啡上线，好喝不贵
})
for (let i = 0; i < 100; i++) {
    eventBus.emit('A店最新咖啡','新品咖啡上线，好喝不贵')
}
```

至此，我们完成了一个简单的发布-订阅模式的代码设计。