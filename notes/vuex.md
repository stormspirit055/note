* ```Vue.use(Vuex)```部分
1. ```Vue.use(Vuex)```: 调用```plugin.install```

>  Vuex 作为```vue```的一个插件, 会通过```Vue.use```去应用他, 其中的主要逻辑就是调用插件内置的```install```方法初始化

2. ```install```: 执行```applyMixin```
3. ```applyMixin```: 主要逻辑是将```vuexInit```注入到```beforeCreated```中
* ```new Store``` 部分

 1. ```this._modules = new ModuleCollection(options)```
 > 完成```module```树的构建. ```ModuleCollection```实例化的过程就是执行了```register```方法. ```register```方法首先通过```new Module```来创建一个```module```实例, ```Module```类是用来描述单个模块的类.```register```中会递归调用```register```方法, 通过```Module.prototype.addChild```建立父子模块间的关系


 2. ```installModule```
 * ```namespace```的作用: 提高模块的封装度和复用性. 通过添加```namespace: true```的方式使其成为带命名空间的模块
 > 首先通过```vue.set```对非root module中的state做初始化处理, 将其变成响应式并触发更新
###### makeLocalContext
 > 安装模块.首先获取```namespace```, 通过```store._modulesNamespaceMap[namespace] = module``` 将module保存到map中, 然后通过```makeLocalContext```构建一个本地上下文```local``` , 一个local有4个属性```dispatch```(派发action),```commit```(派发mutation), ```getters```(计算state), ```state```, 其中```getters```和```state```需要通过```Object.defineProperties```劫持其```get```, 因为这两个属性在所依赖的属性会在程序进行时修改. 共同逻辑(不包括state)是封装一层结合了```namespace```的```type```,```state```的实现是通过```path.reduce```循环查找**目标state**

###### registerMutation
> 将拼接好的```path```(例如moduleA下的changeIndex => moduleA/changeIndex)作为```key```, 存储在```store._mutation```中

###### registerAction
> 与```registerMutation```相比, 主要有两处不一致, 1. 入参不一致 2. 多了异步处理, 并会返回一个```Promise```对象

###### registerGetter
> 将拼接好的```path```作为```key```存储在```store._wrappedGetters```, 增加了4个入参


###### 最后
> 遍历所有子 modules，递归执行```installModule``` 方法

 3. ```resetStoreVM```
 * 作用: 建立 getters 和 state 的联系, 并通过Vue中```computed```计算属性实现了对```getter```的缓存. 同时响应式注册了state



##### 严格模式
* Store类有一个```strict```参数, 用来控制Vuex的严格模式, 在严格模式中, state的修改必须通过显示的mutation操作, 否则会报错.
###### 具体实现
>  在严格模式中, ```store._vm```会添加一个```watcher```来观测```this._data.$$state```的变化, 并用assert断言来检测store._committing是否为true, 若为false就会触发断言抛出异常.在```_withCommit```方法中,每次显示提交mutation都会将```store._committing```置为true