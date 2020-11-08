### Vuex

Vuex **集中式** 存储管理应用的所有组件的状态，并以相应的规则保证状态以 **可预测** 的方式发生变化。

集中式：整个应用程序状态集中在一个store中保存

可预测：所有使用数据的人都不知道数据哪来的，修改数据也不能直接改，需要使用别人提供的修改方法

![vuex单向数据](https://vuex.vuejs.org/vuex.png)

场景：

- SSR原理，在服务器上跑一个Vuex，做一个快照把store中的状态渲染出来，然后把生成这个快照的html返回给前端
- 日志记录
- 统计分析

Vuex本质还是通过响应式数据触发视图重新渲染，Redux是通过发布-订阅模式实现，这是两者重要的区别

#### 核心概念

- state 状态、数据
- mutations 更改状态的函数
- actions 异步操作
- store 包含以上概念的容器

#### 状态 - state

state保存应用状态

```js
export default new Vuex.Store({
  state: { counter:0 },
})
```

#### 状态变更 - mutations

mutations用于修改状态，store.js

```js
export default new Vuex.Store({
  mutations: {
   add(state) {
     state.counter++
    }
  }
})
```

#### 派生状态 - getters

从state派生出新状态，类似计算属性

```js
export default new Vuex.Store({
  getters: {
    doubleCounter(state) { // 计算剩余数量
      return state.counter * 2;
    }
  }
})
```

#### 动作 - actions

添加业务逻辑，类似于controller

```js
export default new Vuex.Store({
  actions: {
    add({ commit }) {
      setTimeout(() => {
        commit('add')
      }, 1000);
    }
  }
})
```

**使用**

```html
<!-- 同步 -->
<p @click="$store.commit('add')">counter: {{$store.state.counter}}</p>
<!-- 异步 -->
<p @click="$store.dispatch('add')">async counter: {{$store.state.counter}}</p>
<!-- 派生 -->
<p>double：{{$store.getters.doubleCounter}}</p>
```

### vuex实现

#### 需求分析

- 实现一个插件：声明Store类，挂载$store
- Store具体实现：
  - 创建响应式的state，保存mutations、actions和getters
  - 实现commit根据用户传入type执行对应mutation
  - 实现dispatch根据用户传入type执行对应action，同时传递上下文
  - 实现getters，按照getters定义对state做派生

#### 初始化：Store声明、install实现，vuex.js：

```js
let Vue

class Store {
  constructor(options = {}) {
    // 响应化处理state
    // this.state = new Vue({ data: options.state })
    // 上面直接返回state不好，用户可以直接访问到state并修改
    this._vm = new Vue({
      data: {
        // 加两个$$，Vue不做代理，不能直接访问到
        $$state:options.state
      }
    })
  }
  
  // 通过存取器访问到state，store.state
  get state() {
    // this._vm.$data.$$state也能访问到
    return this._vm._data.$$state
  }
  
  // 这里还是无法阻止用户直接修改state的，官方实现是通过watch，监听到任何修改直接报错的
  // 但可以改state里面的值
  set state(v) {
    console.error('please use replaceState to reset state')
  }
}

function install(_Vue) {
  Vue = _Vue;
  Vue.mixin({
    beforeCreate() {
      // 挂载$store方法及通过mixin挂载原因和vue-router一样
      if (this.$options.store) {
        Vue.prototype.$store = this.$options.store
      }
    }
  })
}

export default { Store, install }
```

#### 实现commit：根据用户传入type获取并执行对应mutation

```js
// 和commit无关代码略去
class Store {
  constructor(options = {}) {
    // 保存用户配置的mutations选项
    this._mutations = options.mutations || {}
  }
    
  commit(type, payload) {
    // 获取type对应的mutation
    const entry = this._mutations[type]
    
    if (!entry) {
      console.error(`unknown mutation type: ${type}`)
      return
    }
      
    // 指定上下文为Store实例
    // 传递state给mutation
    entry(this.state, payload)
  }
}
```

#### 实现actions：根据用户传入type获取并执行对应mutation

```js
// 和actions无关代码略去
class Store {
  constructor(options = {}) {
    // 保存用户编写的actions选项
    this._actions = options.actions || {}
      
    // 绑定commit上下文否则action中调用commit时可能出问题!!
    // 同时也把action绑了，因为action可以互调
    const store = this
    const {commit, action} = store
    this.commit = function boundCommit(type, payload) {
      commit.call(store, type, payload)
    }
    this.action = function boundAction(type, payload) {
      return action.call(store, type, payload)
    }
    // 用bing也行
    // this.commit = this.commit.bind(this)
    // this.action = this.action.bind(this)
  }
    
  dispatch(type, payload) {
    // 获取用户编写的type对应的action
    const entry = this._actions[type]
    if (!entry) {
      console.error(`unknown action type: ${type}`);
      return
    }
    // 异步结果处理常常需要返回Promise
    return entry(this, payload);
  }
}
```
