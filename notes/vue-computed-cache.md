1. 初始化计算属性(computed)Watcher时, 会传一个参数 ```{lazy: true}```
```
var computedWatcherOptions = { lazy: true };
watchers[key] = new Watcher(
    vm,
    getter || noop,
    noop,
    computedWatcherOptions
  );
 // Watcher 类
function Watcher(..., options) {
    ...
    this.lazy = !!options.lazy
    this.dirty = this.lazy
}
```
2. 每次收到更新通知时, 订阅者都会将```this.dirty``` 置为```true```
```
Watcher.prototype.update = function update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true;
  } else if (this.sync) {
    this.run();
  } else {
    queueWatcher(this);
  }
};
```
3. 每次需要用到计算属性时, 会触发```createComputedGetter```(通过```Object.defineProperty```劫持了计算属性的```get```)
```
function createComputedGetter (key) {
  return function computedGetter () {
    var watcher = this._computedWatchers && this._computedWatchers[key];
    if (watcher) 
    //如果是脏数据, 会重新计算
      if (watcher.dirty) {
        console.log('hahaha')
        watcher.evaluate();
      }
      if (Dep.target) {
        watcher.depend();
      }
      console.log(watcher.value)
      return watcher.value
    }
  }
}
```




