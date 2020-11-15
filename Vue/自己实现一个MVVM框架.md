### 理解Vue的设计思想

MVVM框架的三要素： **数据响应式、模板引擎及其渲染**

数据响应式：监听数据变化并在视图中更新

- Object.defineProperty()
- Proxy

模版引擎：提供描述视图的模版语法

- 插值：{{}}
- 指令：v-bind，v-on，v-model，v-for，v-if

渲染：如何将模板转换为html

- 模板 => vdom => dom

### 数据响应式原理


数据变更能够响应在视图中，就是数据响应式。vue2中利用Object.defineProperty()实现变更检

测。

#### 简单实现

```js
const obj = {}

function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      console.log(`get ${key}:${val}`)
      return val
    },
    set(newVal) {
      if (newVal !== val) {
        console.log(`set ${key}:${newVal}`)
        val = newVal
      }
    }
  })
}

defineReactive(obj, 'foo', 'foo')
obj.foo
obj.foo = 'foooooooooooo'
```

**结合视图**

```html
<!DOCTYPE html>
<html lang="en">
<head></head>
<body>
  <div id="app"></div>
  <script>
    const obj = {}
    
    function defineReactive(obj, key, val) {
      Object.defineProperty(obj, key, {
        get() {
          console.log(`get ${key}:${val}`)
          return val
        },
        set(newVal) {
          if (newVal !== val) {
            val = newVal
            update()
          }
        }
      })
    }
      
    defineReactive(obj, 'foo', '')
    obj.foo = new Date().toLocaleTimeString()
      
    function update() {
      app.innerText = obj.foo
    }
      
    setInterval(() => {
      obj.foo = new Date().toLocaleTimeString()
    }, 1000);
  </script>
</body>
</html>
```
#### 遍历需要响应化的对象

```js
// 对象响应化：遍历每个key，定义getter、setter
function observe(obj) {
  if (typeof obj !== 'object' || obj == null) {
    return
  }
  Object.keys(obj).forEach(key => {
    defineReactive(obj, key, obj[key])
  })
}

const obj = {foo:'foo',bar:'bar',baz:{a:1}}
observe(obj)
obj.foo
obj.foo = 'foooooooooooo'
obj.bar
obj.bar = 'barrrrrrrrrrr'
obj.baz.a = 10 // 嵌套对象no ok
```

#### 解决嵌套对象问题

```js
function defineReactive(obj, key, val) {
  // 每次defineReactive时再对每个val执行observe
  observe(val)
  Object.defineProperty(obj, key, {
    // ...
```
#### 解决赋的值是对象的情况

```js
obj.baz = {a:1}
obj.baz.a = 10 // no ok

function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      console.log(`get ${key}:${val}`)
      return val
    },
    set(newVal) {
      if (newVal !== val) {
        observe(newVal) // 新值是对象的情况，对每个新值执行observe
        val = newVal
        update()
      }
    }
  })
}
```

#### 如果添加/删除了新属性无法检测

```js
obj.dong = 'dong'
obj.dong // 没有触发get()

// 添加一个set方法来添加新属性
// Vm.$set
function set(obj, key, val) {
  defineReactive(obj, key, val)
}

set(obj, 'dong', 'dong')
```

### Vue中的数据响应化

#### 目标代码

vue.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF- 8 ">
  <meta name="viewport" content="width=device-width, initial-scale= 1. 0 ">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
</head>
<body>
  <div id="app">
    <p>{{counter}}</p>
  </div>
  <script src="./vue.js"></script>
  <script>
    const app = new Vue({
      el:'#app',
      data: {
        counter: 1
      },
    })
    
    setInterval(() => {
      // app.$data.counter++
      // 通过proxy函数将$data中的key代理到vm属性中，可以直接vm.key访问$data数据
      app.counter++
      
    }, 1000);
  </script>
</body>
</html>
```

#### 原理分析

1. new Vue()首先执行初始化，对data执行响应化处理，这个过程发生在Observer中
2. 同时对模板执行编译，找到其中动态绑定的数据，从data中获取并初始化视图，这个过程发生在
   Compile中
3. 同时定义一个更新函数和Watcher，将来对应数据变化时Watcher会调用更新函数
4. 由于data的某个key在一个视图中可能出现多次，所以每个key都需要一个管家Dep来管理多个
   Watcher
5. 将来data中数据一旦发生变化，会首先找到对应的Dep，通知所有Watcher执行更新函数

![image-20201108222022613](C:\Users\byche\AppData\Roaming\Typora\typora-user-images\image-20201108222022613.png)

> 解析指令：模板里面出现的非HTML的东西，比如`{{}}`数据绑定，特殊指令`v-if`、`v-model`、`@click`等
>
> 观察者Watcher：视图中出现一个数据绑定就创建一个Watcher（同一个数据出现多次会创建多个Watcher）并保存一个更新函数，数据变化时就执行
>
> Dep：每个属性和Dep是一对一关系，Dep和Watcher是一对多关系

**涉及类型介绍**

- Vue：框架构造函数
- Observer：执行数据响应化（分辨数据是对象还是数组）
- Compile：编译模板，初始化视图，收集依赖（更新函数、watcher创建）
- Watcher：执行更新函数（更新dom）
- Dep：管理多个Watcher，批量更新

#### Vue

##### 框架构造函数：执行初始化

执行初始化，对data执行响应化处理，vue.js

```js
function observe(obj) {
  if (typeof obj !== 'object' || obj == null) {
    return
  }
  
  // 创建Observer实例
  new Observer(obj)
}

// 代码同上面数据响应式原理章节推导出来的defineReactive方法
function defineReactive(obj, key, val) {
  observe(val) // 解决嵌套对象问题
  
  Object.defineProperty(obj, key, {
    get() {
      console.log(`get ${key}:${val}`)
      return val
    },
    set(newVal) {
      if (newVal !== val) {
        observe(newVal) // 解决赋的值是对象的情况
        val = newVal
      }
    }
  })
}

class Vue {
  constructor(options) {
    this.$options = options
    this.$data = options.data
    
    // 响应化处理
    observe(this.$data)
    
    // 代理 (实现代码在下面为$data做代理章节)
    proxy(this, '$data')
    
    // 创建编译器 (实现代码在下面编译-Compile章节)
    new Compiler(options.el, this)
  }
}

// 根据对象类型决定如何做响应化
class Observer {
  constructor(value) {
    this.value = value
    this.walk(value)
  }
  
  // 对象数据响应化
  walk(obj) {
    Object.keys(obj).forEach(key => {
      defineReactive(obj, key, obj[key])
    })
  }
  
  // TODO 数组数据响应化
  /**
   * // 获取数组原型对象
   * const arrayProto = Array.prototype // 原型用
   * // 复制一份
   * const arrayMethods = Object.create(arrayProto)
   * const methodsToPatch = ['push','pop','shift','unshift','splice','sort','reverse']
   * methodsToPatch.forEach(function (method) {
   *   arrayMethods[method] = function() {
   *     // 原始操作
   *     arrayProto[method].apply(this. arguments)
   *     // 追加操作：通知更新
   *     // dep.notify()
   *   }
   * }
   *
   * function observe(obj) {
   *   // ...
   *   // 判断是数组时，替换原型为上面修改过的
   *   obj.__proto__ = arrayProto // 实例的原型通过__proto__修改
   * }
   */
}
```

为$data做代理，vue.js

```js
class Vue {
  constructor(options) {
    // ...
    proxy(this, '$data')
  }
}

// 代理函数，方便用户直接访问$data中的数据
// 不代理的话用户访问$data的数据需要  vm.$data.foo
// 代理后  vm.foo
function proxy(vm, sourceKey) {
  // vm[sourceKey]就是vm['$data'] 传的sourceKey值为'$data'
  Object.keys(vm[sourceKey]).forEach(key => {
    // 将$data中的key代理到vm属性中
    Object.defineProperty(vm, key, {
      get() {
        return vm[sourceKey][key]
      },
      set(newVal) {
        vm[sourceKey][key] = newVal
      }
    })
  })
}
```

##### 编译 - Compile

编译模板中vue模板特殊语法，初始化视图、更新视图

![image-20201114143425423](C:\Users\byche\AppData\Roaming\Typora\typora-user-images\image-20201114143425423.png)

递归遍历DOM树 => 判断节点类型，如果是文字则判断是否插值绑定，如果是元素则遍历其属性判断是否指令或事件，然后递归子元素

**初始化视图**

根据节点类型编译，compile.js

```js
class Compiler {
  // el：宿主元素
  // vm：Vue实例
  constructor(el, vm) {
    this.$vm = vm
    this.$el = document.querySelector(el)
    if (this.$el) {
      // 执行编译
      this.compile(this.$el)
    }
  }
  
  compile(el) {
    // 遍历el树
    const childNodes = el.childNodes
    // childNodes是一个类数组，最好是from一下
    Array.from(childNodes).forEach(node => {
      // 判断是否是元素
      if (this.isElement(node)) {
        // 编译元素
        this.compileElement(node) // 实现代码在下面章节
      } else if (this.isInterpolation(node)) {
        // 编译插值文本
        this.compileText(node) // 实现代码在下面章节
      }
      
      // 递归子节点
      if (node.childNodes && node.childNodes.length > 0) {
        this.compile(node)
      }
    });
  }
  
  isElement(node) {
    return node.nodeType == 1
  }
  
  isInterpolation(node) {
    // 首先是文本，其次内容是{{xxx}}，正则上加了分组是为了后面可以直接通过RegExp.$1拿到{{}}里的值
    return node.nodeType == 3 && /\{\{(.*)\}\}/.test(node.textContent)
  }
}
```
编译插值，compile.js

```js
class Compiler {
  // ...
  compile(el) {
    // ...
        } else if (this.isInerpolation(node)) {
          // console.log("编译插值文本" + node.textContent);
          this.compileText(node)
        }
    });
  }

  compileText(node) {
    // 直接通过RegExp.$1拿到{{}}里面的key，然后得到Vue.$data中这个key的值
    // 没有考虑{{}}里面有其他更复杂的情况，不单单只是一个$data变量，可能还有函数、表达式等
    node.textContent = this.$vm[RegExp.$1]
  }
  // ...
}
```

编译元素，compile.js

```js
class Compiler {
  // ...
  compile(el) {
    // ...
    if (this.isElement(node)) {
      // console.log("编译元素" + node.nodeName);
      this.compileElement(node)
    }
    // ...
  }

  compileElement(node) {
    // 节点是元素
    // 遍历其属性列表，处理特殊指令 v-xxx
    let nodeAttrs = node.attributes
    Array.from(nodeAttrs).forEach(attr => {
      // v-xxx="oo"
      let attrName = attr.name // v-xxx
      let exp = attr.value // oo
      if (this.isDirective(attrName)) {
        let dir = attrName.substring(2) // xxx
        // 执行指令
        this[dir] && this[dir](node, exp)
      }
    })
  }

  isDirective(attr) {
    return attr.indexOf("v-") == 0
  }

  // v-text
  text(node, exp) {
    node.textContent = this.$vm[exp]
  }
  // v-html
  html(node, exp) {
    node.innerHTML = this.$vm[exp]
  }
  // ...
}
```

##### 依赖收集

视图中会用到data中某key，这称为依赖。同一个key可能出现多次，每次都需要收集出来用一个Watcher来维护它们，此过程称为依赖收集。

多个Watcher需要一个Dep来管理，需要更新时由Dep统一通知

看下面案例，理出思路：

```js
new Vue({
  template:
    `<div>
      <p>{{name 1 }}</p>
      <p>{{name 2 }}</p>
      <p>{{name 1 }}</p>
    <div>`,
  data: {
    name 1 : 'name 1 ',
    name 2 : 'name 2 '
  }
})
```



![image-20201114155119175](C:\Users\byche\AppData\Roaming\Typora\typora-user-images\image-20201114155119175.png)

每个属性和Dep是一对一关系，Dep和Watcher是一对多关系，Dep和Watcher是典型的观察者模式，Dep是被观察者（key和Dep是一对一，key变化会通知Dep，实际被观察的是key），Watcher是观察者

**实现思路**

1. defineReactive时为每一个key创建一个Dep实例
2. 初始化视图时读取某个key，例如name1，创建一个watcher
3. 由于触发name1的getter方法，便将watcher1添加到name1对应的Dep中
4. 当name1更新，setter触发时，便可通过对应Dep通知其管理所有Watcher更新

![image-20201108222022613](C:\Users\byche\AppData\Roaming\Typora\typora-user-images\image-20201108222022613.png)

创建Watcher，vue.js

```js
// 监听器：负责更新视图
// 观察者：保存更新函数，值发生变化调用更新函数
class Watcher {
  constructor(vm, key, updateFn) {
    // vue实例
    this.vm = vm
    // 依赖key
    this.key = key
    // 更新函数
    this.updateFn = updateFn
  }
  
  // 更新
  update() {
    this.updateFn.call(this.vm, this.vm[this.key])
  }
}
```

编写更新函数、创建watcher，compile.js

```js
class Compiler {
  // ...
  
  update(node, exp, dir) {
    // 指令对应的更新函数
    const fn = this[dir+'Updater']
    fn && fn(node, this.$vm[exp])
    
    // 每次访问到一个key就创建一个对应的watcher，供后面统一更新内容
    new Watcher(this.$vm, exp, function(val){
      fn && fn(node, val)
    })
  }
  
  textUpdater(node, val) {
    node.textContent = val
  }

  htmlUpdater(node, val) {
    node.innerHTML = val
  }
  
  // 调用update函数执插值文本赋值
  compileText(node) {
    // console.log(RegExp.$1);
    // node.textContent = this.$vm[RegExp.$1];
    this.update(node, RegExp.$1, 'text')
  }
  
  // 遍历出指令后，在每个指令调用update函数
  compileElement(node) {
    // 节点是元素
    // 遍历其属性列表，处理特殊指令 v-xxx
    let nodeAttrs = node.attributes
    Array.from(nodeAttrs).forEach(attr => {
      // v-xxx="oo"
      let attrName = attr.name // v-xxx
      let exp = attr.value // oo
      if (this.isDirective(attrName)) {
        let dir = attrName.substring(2) // xxx
        // 执行指令
        this[dir] && this[dir](node, exp)
      }
    })
  }
  
  // v-text
  text(node, exp) {
    // node.textContent = this.$vm[exp]
    this.update(node, exp, 'text')
  }
  // v-html
  html(node, exp) {
    // node.innerHTML = this.$vm[exp]
    this.update(node, exp, 'html')
  }
}
```

声明Dep，vue.js

```js
// Dep：依赖，管理某个key相关所有Watcher实例
class Dep {
  constructor () {
    // 保存所有依赖
    this.deps = []
  }
  
  addDep (dep) {
    // push的是每一个Watcher实例
    this.deps.push(dep)
  }
  
  notify() {
    this.deps.forEach(dep => dep.update())
  }
}

```

创建watcher时触发getter

```js
class Watcher {
  constructor(vm, key, updateFn) {
    // ...
    
    // 把当前watcher实例挂到Dep.target静态属性上
    Dep.target = this;
    // 读取一次这个key，触发getter
    // 在getter中会把这个watcher实例添加到这个key对应的Dep实例上(下一章节实现)
    this.vm[this.key];
    // 触发getter添加实例后把target挂载去掉
    Dep.target = null;
  }
  // ...
}
```

依赖收集，创建Dep实例，vue.js

```js
// 在defineReactive方法中
function defineReactive(obj, key, val) {
  observe(val)
  
  // 每定义一个响应式属性就创建一个Dep，Dep和key一一对应
  const dep = new Dep()
  
  Object.defineProperty(obj, key, {
    get() {
      // 收集依赖
      Dep.target && dep.addDep(Dep.target)
      return val
    },
    set(newVal) {
      if (newVal !== val) {
        observe(newVal)
        val = newVal
        // 通知这个key下的watcher更新
        dep.notify()
      }
    }
  })
}

defineReactive(obj, key, val) {
  this.observe(val);
  
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    get() {
      
      return val
    },
    set(newVal) {
      if (newVal === val) return
      dep.notify()
    }
  })
}
```

##### 总结

![image-20201108222022613](C:\Users\byche\AppData\Roaming\Typora\typora-user-images\image-20201108222022613.png)

当创建构造函数new MVVM()时只做两件事，创建Observer实例以及创建Compiler实例

Observer实例：通过`Object.defineProperty`的getter/setter方法劫持了所有属性(key)，defineReactive时为每一个key创建一个Dep实例，在getter方法中收集视图中每次使用key时创建的Watcher实例（Watcher实例在Compiler中创建），当监听到变化时在setter方法中每个key唯一对应的Dep会通知该Dep下的所有Watcher通过更新函数更新视图

Compiler实例：编译模板中Vue模板特殊语法，比如指令：`v-html` `v-text`，插值：`{{}}`，编译该语法为JS语法去初始化视图（其实就是匹配语法对应的更新函数并执行一遍）并创建Watcher实例及传入该语法的更新函数，供后续属性变化时执行

##### 缺陷

这是Vue1.0的实现方法，但有一个致命缺陷是：随着应用的扩大，页面中每个属性的watcher绑定会大量出现，创建的watcher达到一定规模后应用就崩掉，Vue1.0的性能问题主要也是这样引起，所以后面2.0就调整了Watcher实例的粒度，由每个key对应一个watcher改为每个组件对应一个watcher，组件内任何一个key发生变化都通知这一个watcher，但怎么知道哪个key发生变化了呢？2.0引入虚拟DOM的概念，通过重新计算虚拟DOM，比较得出哪个key变化了