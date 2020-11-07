### vue-router

Vue Router 是 Vue.js 官方的路由管理器。它和 Vue.js 的核心深度集成，让构建单页面应用变得易如反

掌。

核心步骤：

- 步骤一：使用vue-router插件，router.js

  ```js
  import Router from 'vue-router'
  Vue.use(Router)
  ```

- 步骤二：创建Router实例，router.js

  ```js
  export default new Router({...})
  ```

- 步骤三：在根组件上添加该实例，main.js

  ```js
  import router from './router'
  new Vue({
    router,
  }).$mount("#app")
  ```

- 步骤四：添加路由视图，App.vue

  ```html
  <router-view></router-view>
  ```

- 导航

  ```html
  <router-link to="/">Home</router-link>
  <router-link to="/about">About</router-link>
  ```

### vue-router源码实现

#### 需求分析

- 作为一个插件存在：实现VueRouter类和install方法
- 实现两个全局组件：router-view用于显示匹配组件内容，router-link用于跳转
- 监控url变化：监听hashchange或popstate事件
- 响应最新url：创建一个响应式的属性current，当它改变时获取对应组件并显示

#### 实现一个插件：创建VueRouter类和install方法

创建vue-router.js

```js
// 引用构造函数，VueRouter中要使用
// 不直接import进来是为了避免引入整个Vue导致插件体积太大，可以直接使用install方法传入的Vue实例
let Vue;

// 保存选项
class VueRouter {
  constructor(options) {
    this.$options = options
  }
}

// 插件：实现install方法，注册$router
VueRouter.install = function(_Vue) {
  // 引用构造函数，VueRouter中要使用
  Vue = _Vue;
  
  // 挂载$router
  Vue.mixin({
    // 通过全局mixin获取到router选项，这是new Vue时传进来的VueRouter实例
    // import Router from './vue-router';
    // Vue.use(Router);
    // const router = new Router({...});
    // new Vue({
    //   router,
    // }).$mount("#app");
    // 这里就可以看到为什么需要用mixin获取VueRouter实例
    // 主要原因是use在前（即执行了install方法），VueRouter实例创建在后，而install逻辑又需要用到该实例
    // 如果我们use完，创建VueRouter实例后，自己再手动挂载到$router也行，vue-router不这样设计可能觉得不够优雅
    beforeCreate() {
      // 全局mixin是会所有组件都会有的，但只有根组件拥有router选项 TODO：如果其他组件也有router呢
      // $options可以获取当前Vue实例/组件的初始化选项
      if (this.$options.router) {
        // vm.$router
        Vue.prototype.$router = this.$options.router
      }
    }
  });
};
export default VueRouter
```

#### 创建router-view和router-link

创建router-link.js

```js
export default {
  props: {
    to: String,
    required: true
  },
  // 注意Runtime-only的Vue只能用render，不能使用template选项直接描述组件，要Runtime+Compiler版本Vue才能用template属性
  // Runtime-only可以在*.vue文件上使用template标签，*.vue文件内部模板会在构建时被webpack的vue-load等工具预编译成JavaScript。你在最终打好的包里实际上是不需要编译器的，所以只用Runtime-only版本即可。
  // 如果你需要在客户端编译模板 (比如传入一个字符串给template选项，或挂载到一个元素上并以其DOM内部的HTML作为模板，即挂载的实例没有template选项也没有render函数)，就将需要加上编译器，即完整版，不在打包时进行编译而是在客户端（浏览器）运行时进行编译的
  // 很显然，这个编译过程对性能会有一定损耗，所以通常我们更推荐使用Runtime-only的Vue
  // template: '<a></a>',
  render(h) {
    // return <a href={'#'+this.to}>{this.$slots.default}</a>;
    // 使用时：<router-link to="/about">xxx</router-link>
    return h('a', {
      attrs: {
          href: '#' + this.to
      }
    }, [
      this.$slots.default
    ])
  }
}
```

创建router-view.js，

```js
export default {
  render(h) {
    // 暂时先不渲染任何内容
    return h(null)
  }
}
```
#### 监控url变化

定义响应式的current属性，监听hashchange事件（哈希版本）

```js
class VueRouter {
  constructor(options) {
    this.$options = options
    /**
     * 监控url变化
     * this.current = '/'
     * window.addEventListener('hashchange', onHashChange)
     * 这样写的话current只是一个没有响应式的普通属性，current值变了没人知道
     */
      
    // 创建响应式的current属性
    // 变成响应式后，任何用到current的组件都会收集起来，当current值变化时就会通知组件作更新
    // 在router-view.js的render函数中使用current获取当前url，当current变化时render函数重新执行从而改变<router-view>标签渲染的component
    // render函数里面使用了响应式数据，当数据变化时都会重新执行render
    // 这里new一个Vue实例来创建响应式数据也行
    // this.app = new Vue({ data() {return {current:'/'} } })
    // 然后这样访问current  this.$router.app.current
    Vue.util.defineReactive(this, 'current', '/')
      
    // 定义响应式的属性current
    const initial = window.location.hash.slice(1) || '/'
    Vue.util.defineReactive(this, 'current', initial)
      
    // 监听hashchange事件
    // 这里需要bind一下，避免onHashChange里面的this指向window
    // 或直接使用箭头函数 () => this.current = window.location.hash.slice(1)
    window.addEventListener('hashchange', this.onHashChange.bind(this))
    // 再监听一下load，获取初次打开页面时的url
    // 这里的效果其实下面获取默认url的效果一样
    // const initial = window.location.hash.slice(1) || '/'
    window.addEventListener('load', this.onHashChange.bind(this))
  }
    
  onHashChange() {
    this.current = window.location.hash.slice(1)
  }
}
```

动态获取对应组件，router-view.js

```js
export default {
  render(h) {
    // 动态获取对应组件，获取path对应的component
    let component = null
    this.$router.$options.routes.forEach(route => {
      if (route.path === this.$router.current) {
        component = route.component
      }
    })
    return h(component)
  }
}
```

#### 提前处理路由表

提前处理路由表避免每次都循环

```js
// vue-router.js
class VueRouter {
  constructor(options) {
    // 缓存path和route映射关系
    this.routeMap = {}
    this.$options.routes.forEach(route => {
      this.routeMap[route.path] = route
    });
  }
}
```

```js
// router-view.js
export default {
  render(h) {
    const {routeMap, current} = this.$router
    const component = routeMap[current].component
    return h(component)
  }
}
```