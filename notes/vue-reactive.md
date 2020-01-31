### 前言
阅读完该文章, 你不一定会掌握响应式原理, 但一定会有助于你掌握响应式原理, 源码这玩意儿如果光看看文章视频, 不自己亲手调试一下的话, 很难掌握. 

### 准备工作
为了方便调试, 我这里调试的不是源码, 而是打包好后的```vue/dist/vue.esm.js```, 这样方便打日志, 也不用切不同的文件. 所以准备工作就是用```vue init webpack vuedemo```初始化一个项目, 然后在```main.js```中初始化一些demo, 如下
```
import Vue from 'vue'
/* eslint-disable no-new */
new Vue({
  el: '#app',
  data(){
    return {
      msg: '天气不错',
    }
  },
  methods: {
    click() {
      //一些逻辑
    }
  },
  template: `
  <div>
    <div>{{msg}}</div>
    <button @click='click'>按钮</button>
  </div>
  `
})

```
ps: 1. 切记不要通过挂载```App```组件的方式调试, 直接用```template```, 如果挂载组件, 会多很多重复的日志, 非常不利于调试 2. 这里只是给大家一些调试的建议(因为一开始我挂了个APP, 调得我好麻烦), 下文中不会出现 **日志** 相关的内容

### 再次前言

本文标题是 **部分**响应式原理, 响应式分三块: ```侦听器(watch)```, ```computed(计算属性)```, ```render(模板渲染)```. 实现响应式的原理都是相同的, 只是在针对特性的业务逻辑上有些不同. 本文就选```render(模板渲染)```的响应式原理具体展开. 接下来会以```调用栈```会主线, 进行原理的分析

### 原理分析
> 为了缩减篇幅, 在一些代码截取上, 我会忽略**报错的警示代码以及与响应式原理无关的逻辑代码**, 通过```...```取代, 不过还是建议大家看下原函数,帮助理解

1. ```initmixin``` 入口函数
```
function initmixin(Vue) {
    Vue.prototype._init = function(options) {
        var vm  = this
        ...
        // 
        initState(vm)
        if (vm.$options.el) { 
            // 这行代码, 在第7步中'呼应1'会解释
            vm.$mount(vm.$options.el);
        }
    }
}
```

2. ```initState(vm)```, 入口函数, 逻辑很简单. 注意: 这里的```opts.data```不是我们写的那个```data(){return {}}```方法, 而是一个name是```mergedInstanceDataFn```的经过包装的方法
```
var opts = vm.$options;
if (opts.data) {
    initData(vm);
} else {
    observe(vm._data = {}, true /* asRootData */);
}
```
3. ```initData(vm)```
```
data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {};
// 这里获取的data, 如果用我给的demo就是{msg: '天气不错'}
var keys = Object.keys(data);
var i = keys.length;
  while (i--) {
    var key = keys[i];
    ...
    if (!isReserved(key)) {
      // isReserved 函数是用来判断 data中的属性是否已 $ 或者 _ 开头, 因为这俩开头的属性可能会和vue内置的属性, API冲突, 所以vue选择不代理他们
      // proxy 只是将data中的属性代理到vm实例上,这样就可以用this.xxx直接获取数据, 要实现响应式还得看下面的observe
      proxy(vm, "_data", key);
    }
  }
  ...
  //观察他们!
  observe(data, true /* asRootData */);
```
4. ```observer(value, asRootData)```
```
if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__;
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
  // 生成一个Observer实例
    ob = new Observer(value);
  }
```
5. ```class Observer``` 声明一个观察者类
```
  if (Array.isArray(value)) {
  // 如果是数组, 则调用observeArray, 其最终还是会走walk
    this.observeArray(value);
  } else {
    this.walk(value);
  }
```
6. ```Observer.prototype.walk```
```
// 遍历每个属性, 并执行defineReactive$$1
Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive$$1(obj, keys[i]);
  }
};
```
7. ```defineReactive$$1```(直译就是定义响应式), 这是很关键的一个函数, 这个函数中定义了每个响应式属性的```getter```& ```setter```, 并在getter中执行**依赖收集**, 在setter中执行**派发更新**. 看这个函数前, 我强烈建议读者打开源码一起往下走, 因为这里特别绕, 如果光看文章, 很难理解
```
// 这里进行了大幅的代码删减, 只为展示最直接的逻辑
function defineReactive$$1(obj, key, val, customSetter, shallow) {
    // Dep是一个依赖类, 看下面代码
    var dep = new Dep()
    Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    //标记①:
    get: function reactiveGetter () {
    // 注①
    // Dep.target 是一个Watcher实例(可理解为一个订阅者), 如果Dep.target 不为undefined, 则去收集依赖
      if (Dep.target) {//
        dep.depend();
      }
      return value
    },
    //标记②:
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      dep.notify(); //派发更新
    }
  });
}
// Dep
var Dep = function Dep () {
  this.id = uid++; // id
  this.subs = [];  // 订阅者数组
};

```
注①:
1. ```Dep.target``` 什么时候被赋值, 它是个什么?
>1.全局搜索```Dep.target```, 会有两个函数中对其进行了赋值, 1. ```pushTarget``` 2. ```popTarget``` 可以看出这是逻辑相反的两个函数, 我们就看```pushTarget``` \
> 2. 全局搜索```pushTarget```, 会发现有5处地方调用了, 但只有```Watcher.prototype.get```中给他传参了, 因为传的是```this```, 所以很明显```Dep.target```是一个Watcher的实例(即订阅者)
```
/**
 * Evaluate the getter, and re-collect dependencies.(翻译: 计算一个getter, 并重新收集依赖)
 */
Watcher.prototype.get = function get () {
  pushTarget(this);
  var value;
  var vm = this.vm;
  try {
    value = this.getter.call(vm, vm);
  } catch (e) {
    ...
  } finally {
    // "touch" every roperty so they are all tracked as
    // dependencies for deep watching
    // watch中 有deep:true 属性的 会进入traverse, 进行递归绑定, 这里我们忽略递归绑定逻辑
    if (this.deep) {
      traverse(value);
    } 
    //这两步目前不用管
    popTarget();
    this.cleanupDeps();
  }
  return value
};
```
>3 找到了给```Dep.target```调用的地方, 也引入了一个```Watcher```的概念, 那系统是什么时候创建的```Watcher```实例的呢?全局搜索```new Watcher```你会发现3个地方用到了, 分别是在```Vue.prototype.$watch```(侦听器)中, ```initComputed```(计算属性)中, 以及```mountComponent```(挂载组件)中(呼应1:```mountComponent```是在```Vue.prototype.$mount```中调用的),所以所有订阅者均来自于这三个地方.本要讲的也就是```mountComponent```时创建的订阅者. 所以如果已我本文开头是给的```demo```为例, 只会生成一个```Watcher```(解释一下: demo中我只订阅了msg一个依赖, 如果我多订阅几个依赖, 依旧是一个Watcher.而如果是侦听器或者计算属性,则会生成对应多个watcher) \
>4 那么```Watcher.prototype.get```是在什么时候调用的呢?在```render Watcher```(就是本文要讲的Watcher, 即模板渲染Watcher)中, 有两个地方调用了, 一个是在```function Watcher```的最下面,这个很明确, 意思就每新建一个```Watcher```实例, 必然会执行一次```Watcher.prototype.get```, 一个是在```Watcher.prototype.run``` 中.这个根据**调用栈** 去倒推, 会发现是这样的调用栈: 1.  触发属性的```set```(标记②) => 2. ```dep.notify``` => 3.```Watcher.prototype.update``` => 4. ```queueWatcher``` => 5. ```flushSchedulerQueue``` => 6. ```watcher.run()```
```
//function Watcher
...
  this.value = this.lazy
    ? undefined
    : this.get();  <= 这个就是调用```Watcher.prototype.get```
// Watcher.prototype.run
...
  if (this.active) {
    var value = this.get();<= 这个就是调用```Watcher.prototype.get```
```
>5 目前为止我们知道了什么时候触发```Watcher.prototype.get```即(Dep.target什么时候是一个Watcher), 这个时候我们还需要搞清楚什么时候触发属性的```get```(标记①).在```Watcher.prototype.get```中
```
Watcher.prototype.get = function get () {
    ...
    try {
        console.log(this.getter)
        value = this.getter.call(vm, vm); <=这一行是触发了属性的get, 可以尝试注释它, 页面就会不渲染
    } catch (e) {
    ...
}
```
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-gold-cdn.xitu.io/2020/1/20/16fc117c5d736e4f?w=448&h=49&f=jpeg&s=5718">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">log的getter</div>
</center>

>6 整理下触发订阅者进行依赖收集(呼应2:或者说依赖进行订阅者收集,后文会讲到)逻辑: 1. 执行模板渲染函数时, 如果有用到依赖属性, 则会触发依赖属性的```get```(比如我页面要渲染```{{msg}}```, 则会触发msg的```get```), 并执行依赖收集 2.当属性的值发生改变时, 会执行dep.notify通知视图层更新, 同样会触发依赖属性的```get```. 我们知道了触发依赖收集的条件, 然后我们研究下属性是如何执行**依赖收集**的
```
//defineReactive$$1
function defineReactive$$1() {
    ...
    Object.defineProperty(obj, key, {
    ...
    get: function reactiveGetter () {
      if (Dep.target) {
        dep.depend();    <= 依赖收集入口
      }
      return value
    },
}
// depend
Dep.prototype.depend = function depend () {
  if (Dep.target) {
    Dep.target.addDep(this);    <= 注意这的Dep.target是一个Watcher
  }
};
// addDep
Watcher.prototype.addDep = function addDep (dep) {
  var id = dep.id;
  if (!this.newDepIds.has(id)) {
  // 订阅者执行依赖收集
    this.newDepIds.add(id);   
    this.newDeps.push(dep);
    if (!this.depIds.has(id)) {
      dep.addSub(this);
    }
  }
};
// addSub
Dep.prototype.addSub = function addSub (sub) {
//呼应2: 依赖执行订阅者收集
  this.subs.push(sub);   
};

```
>7 收集完订阅者, 来看看如何通知订阅者完成**派发更新**的
```
function defineReactive$$1(obj, key, val, customSetter, shallow) {
    ...
    Object.defineProperty(obj, key, {
    ...
    set: function reactiveSetter (newVal) {
      ...
      dep.notify(); //派发更新
    }
  });
}
// notify
Dep.prototype.notify = function notify () {
  var subs = this.subs.slice();
  if (process.env.NODE_ENV !== 'production' && !config.async) {
    // 对订阅者排序
    subs.sort(function (a, b) { return a.id - b.id; });
  }
  for (var i = 0, l = subs.length; i < l; i++) {
  // 呼应2: 挨个通知订阅者该更新了. 这也是为什么为了便于理解,我偏向于叫订阅者收集,
  // 因为他派发更新的主逻辑是,依赖收集订阅者,然后依赖挨个通知订阅者
    subs[i].update();
  }
};

```

>8 ```subs[i].update()```后还有一系列逻辑, 主要就是```queueWatcher``` => ```flushSchedulerQueue``` => ```Watcher.prototype.run``` => ```Watcher.prototype.get``` => ```this.getter.call(vm, vm);```执行渲染 并触发属性中```get```

以上就是实现**响应式**的一系列最简洁的逻辑
#### 总结一下
1. Dep类是一个依赖类, 有一个自增id属性和一个订阅者数组属性
2. Watcher类是一个订阅者类, 有三种类型: 渲染Watcher, 侦听器Watcher, 计算属性Watcher.
3. 触发依赖收集有两种逻辑,1. 每个新建Watcher, 都会触发订阅者收集相关依赖 2. 当收到**派发更新**通知时, 会更新视图层, 并触发相关依赖的```get```, 并然后重新收集依赖(re-collect dep). 但本质都是触发渲染, 收集相关依赖

#### 深入一下

1. 在```Watcher.prototype.get```中有一步是```cleanupDeps()```
```
Watcher.prototype.get = function get () {
  console.log('执行watcherget')
  pushTarget(this);
  var value;
  var vm = this.vm;
  try {
    ...
  } catch (e) {
    ...
    ...
    this.cleanupDeps();  // 这里
  }
  return value
};
// cleanupDeps
Watcher.prototype.cleanupDeps = function cleanupDeps () {
  var i = this.deps.length;
  while (i--) {
    var dep = this.deps[i];
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this);
    }
  }
  var tmp = this.depIds;
  this.depIds = this.newDepIds;
  this.newDepIds = tmp;
  this.newDepIds.clear();
  tmp = this.deps;
  this.deps = this.newDeps;
  this.newDeps = tmp;
  this.newDeps.length = 0;
};
```
显然这一步是为了清洗依赖, 什么时候需要清洗依赖?
```
<template>
    // 手动的将v-if置为false时, 原本需要订阅的msg,就无需再订阅了, 这也是cleanupDeps的作用
    <div v-if=false>{{msg}}</div>
</template>
```

#### 最后
以上是我总结的```响应式原理```的最简化逻辑, 其实有很多需要拓展的分支, 但要通过文章媒介实在过于麻烦. 所以我觉得想要学习源码的最好途径就是去debugger