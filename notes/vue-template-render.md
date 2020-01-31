### vue模板渲染逻辑

![alt 属性文本](https://user-gold-cdn.xitu.io/2020/1/31/16ffc06687c641de?w=1012&h=752&f=jpeg&s=150397)
1. ```vue.prototype.$mount```: (源码中声明了两个该函数, 第一个被全局的声明的mount存下来了), 调用```compileToFunctions```
2. ```compileToFunctions```: 主要是讲```template```编译成```render```函数. 首先会读缓存(```cache[key]```),没用缓存就会调用```compile```函数拿到render function的字符串形式, 再通过```createFunction```生成render function
3. ```compile```函数:主要分为三个步骤 ```parse```, ```optimize```, ```generate```, 最终输出一个包含ast, render, staticRenderFns的对象
* ```parse```: 将template字符串解析成ast
* ```optimize```: 标记静态节点, 为后面的patch过程中对比新旧VNODE树做优化, 被标记为static的节点在后面的diff算法中会直接忽略
* ```generate```: 根据ast结构拼接成render function的字符串
4. 生成render function后, 会将该函数存到vm.$options.render
5. ```mount```(即第一个vue.prototype.$mount): 这里调用了```mountComponent```
6. ```mountComponent```:执行了```vm._render```(里面调用的是之前生成的```render```),然后调用了```Vue.prototype._update```, 并将```render```生成的```vnode```传入
7. ```Vue.prototype._update```: 进行```__patch__```, ```patchNode```, ```updateChildren```(diff算法高效的核心, 维护4个变量, oldStartIdx, oldEndIdx, newStartIdx,newEndIdx)  
8. ```insert```: 插入元素, 完成渲染