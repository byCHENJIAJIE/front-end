### 虚拟DOM

#### 概念

虚拟DOM（Virtual DOM）是对DOM的JS抽象表示，它们是JS对象，能够描述DOM结构和关系。应用的各种状态变化会作用于虚拟DOM，最终映射到DOM上。

![image-20201118220545750](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201118220545750.png)

体验虚拟DOM

```html
<!DOCTYPE html>
<html lang="en">
<head></head>
  
<body>
  <div id="app"></div>
  <!--安装并引入snabbdom，获取patch函数和h函数，Vue也是用这个-->
  <script src="node_modules/snabbdom/dist/snabbdom.js"></script>
  <script>
    const obj = {}
    // patch函数：对比两个虚拟DOM，执行DOM操作
    // h函数：创建虚拟DOM（render函数传入的参数就是这个h函数，得到虚拟DOM）
    const { init, h } = snabbdom;
    const patch = init([])
    // 保存旧的vnode
    let vnode;
    function defineReactive(obj, key,web , val) {}

    // 更新
    function update() {
      // 修改为patch方式做更新，避免了直接接触dom
      vnode = patch(vnode, h('div#app', obj.foo))
    }
    
    defineReactive(obj, 'foo', new Date().toLocaleTimeString())
    
    // 初始化
    // 如果patch第一个参数是真实DOM，则不会比较直接用第二个参数的虚拟DOM转换给真实DOM替换第一个参数
    // 返回虚拟DOM，patch里面通过比较得出操作最少的方式更新DOM
    vnode = patch(app, h('div#app', obj.foo))
    console.log(vnode);
    setInterval(() => {
      obj.foo = new Date().toLocaleTimeString()
    }, 1000);
  </script>
</body>
</html>
```

#### 优点

- 轻量、快速：DOM操作的执行速度远不如Javascript的运算速度快，操作原生DOM慢，JS运行效率高，将DOM对比操作放在JS层，提高效率，当它们发生变化时通过新旧虚拟DOM比对可以得到最小DOM操作量，从而提升性能

  ```js
  patch(vnode, h('div#app', obj.foo))
  ```

- 跨平台：虚拟DOM 是以 JavaScript 对象为基础而不依赖真实平台环境，所以使它具有了跨平台的能力，比如说浏览器平台、Weex、Node 等，虚拟DOM更新也可以转换为不同运行时特殊操作实现跨平台

  ```js
  // init时传入不同平台的配置：snabbdom_style.default
  const patch = init([snabbdom_style.default])
  patch(vnode, h('div#app', {style:{color:'red'}}, obj.foo))
  ```

- 兼容性：还可以加入兼容性代码增强操作的兼容性

- 提升渲染性能：虚拟DOM的优势不在于单次的操作，而是在大量、频繁的数据更新下，能够对视图进行合理、高效的更新

#### 必要性

vue 1.0中有细粒度的数据变化侦测，它是不需要虚拟DOM的，但是细粒度造成了大量开销，这对于大型项目来说是不可接受的。因此，vue 2.0选择了中等粒度的解决方案，每一个组件一个watcher实例，这样状态变化时只能通知到组件，再通过引入虚拟DOM去进行比对和渲染。

#### 整体流程

watcher.run() => componentUpdated() => render() => update() => patch()

一个数据发生变化，watcher被添加到更新队列中，下一个"Tick"时开始执行更新队列，`watcher.run`被触发执行组件更新函数`componentUpdated`，里面调用`render方法`计算最新虚拟DOM，然后把虚拟DOM传到`update方法`里面去做更新，update里面去做patch，`patch方法`根据参数不同执行初始化或者更新

```js
// src/core/instance/lifecycle.js
// 用户$mount()时定义updateComponent，并创建一个watcher传入更新函数，watcher后面执行run时就会执行updateComponent
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...

  let updateComponent
  updateComponent = () => {
    // 重新计算虚拟DOM，重新执行更新
    vm._update(vm._render(), hydrating)
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    // ...
  }, true /* isRenderWatcher */)
  
  // ...
}

// 更新函数
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  // 上一次更新的虚拟DOM
  const prevVnode = vm._vnode
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // 没有prevVnode则初始化
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 有就执行更新
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  
  // ...
}
```

#### patch获取

patch是createPatchFunction的返回值，传递nodeOps和modules是web平台特别实现

```js
// 传入平台特有的节点操作选项给工厂函数，返回patch方法
// 工厂函数：一个最后返回值是对象的函数，但它既不是类，也不是构造函数
// nodeOps：DOM操作相关方法
// modules：属性、样式、事件、类等操作相关方法
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

#### patch实现

首先进行树级别比较，可能有三种情况：增删改。

- new VNode不存在就删
- old VNode不存在就增
- 都存在就执行diff执行更新

![image-20201118220434135](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201118220434135.png)

```js
// src/core/vdom/patch.js
export function createPatchFunction (backend) {
  // 平台特有属性、节点操作
  const { modules, nodeOps } = backend
  // ...
  
  // 这里return patch而不是直接声明patch为了跨平台，这里的patch是通用方法, 但每个平台对属性、节点操作有差异，其他特定平台调用时传入不同操作方法
  /**
   * // src/platforms/web/runtime/patch.js
   * // 工厂函数返回web平台特有的patch
   * export const patch: Function = createPatchFunction({ nodeOps, modules })
   */
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    // 新节点不存在，删
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    // 老节点不存在，增
    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      // 都存在
      // 若传入的是真实节点，初始化
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        // 存在新老节点，patch算法发生的地方，执行diff
        // patchVnode细节看下面章节
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        // 真实节点，初始化过程，创建新dom树并追加，删除老的模板
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        // 获取老dom
        const oldElm = oldVnode.elm
        // 获取父元素，body
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        // 创建真实dom树
        // 一次性创建好真实DOM树并放到放到老节点后面，减少DOM操作
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm) // 放到老节点后面，和老节点同级，这个时候视图能看到新老两个节点同时存在
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        // 删除老节点树
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

##### patchVnode

比较两个VNode，包括三种类型操作：**属性更新、文本更新、子节点更新**

深度优先，同级比较具体规则如下：

1. 新老节点**均有children**子节点，则对子节点进行diff操作，调用**updateChildren**
2. 如果**老节点没有子节点而新节点有子节点**，先清空老节点的文本内容，然后为其新增子节点
3. 当**新节点没有子节点而老节点有子节点**的时候，则移除该节点的所有子节点
4. 当**新老节点都无子节点**的时候，只是文本的替换

第一点中会递归到新节点或者老节点没有子节点那一个VNode再进行比较，体现深度优先

当两个节点不一样但是他们的子节点一样怎么办？diff是同级比较的，如果第一层不一样那么就不会继续深入比较第二层了，相同子节点不能重复利用，算是一个缺点

```js
// src/core/vdom/patch.js
export function createPatchFunction (backend) {
  // ...
  // 平台特有属性、节点操作
  const { modules, nodeOps } = backend

  // 把这些操作存入cbs，未来在patch节点时统一调用
  // 全局定义了const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = [] // cbs['create'] = []
    for (j = 0; j < modules.length; ++j) {
      // 如果传入modules存在对应钩子函数
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]]) // cbs['create'] = [attrFn, classFn]
      }
    }
  }
  
  
  // 比对新旧vnode
  function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    // ...一些优化方法，跳过diff

    // 传入数据，执行一些组件钩子
    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    // 获取两个节点孩子
    const oldCh = oldVnode.children
    const ch = vnode.children

    // 属性更新 <div style="color:blue"> => <div style="color:red">
    // 如果有data，证明可能会有属性更新
    if (isDef(data) && isPatchable(vnode)) {
      // 在cbs中关于属性更新的数组拿出来 [attrFn, classFn, ...]
      // 遍历全执行一遍，不管有没有变化
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }

    // 判断是否元素，虚拟DOM规定节点没有文本，则是元素
    if (isUndef(vnode.text)) {
      // 都有孩子，比孩子  
      if (isDef(oldCh) && isDef(ch)) {
        // reorder重排，细节看下一章节
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        // 只有新的有孩子，创建孩子
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        // 防止老节点有文本，清空老节点文本
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        // 创建孩子并追加
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 只有老的有孩子，删除所有孩子
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 都没有孩子，清空老节点文本
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 都是文本，更新文本
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
}
```

##### updateChildren

updateChildren主要作用是用一种较高效的方式比对新旧两个VNode的children得出最小操作补丁，执行一个双循环（遍历循环两个数组，一一拿出比较）是传统方式，vue中针对web场景特点做了特别的算法优化，减少循环次数：一般的节点操作都是从后面追加或者从前面插入，很少会从中间插入节点，所以先比较几个特定的边界情况，不符合再遍历比较

![image-20201122154458276](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122154458276.png)

在新老两组VNode节点的左右头尾两侧都有一个变量标记，在**遍历过程中这几个变量都会向中间靠拢**， 当**oldStartIdx > oldEndIdx**或者**newStartIdx > newEndIdx**时结束循环

下面是遍历规则：

首先，oldStartVnode、oldEndVnode与newStartVnode、newEndVnode**两两交叉比较**，共有4种比较方法

当 oldStartVnode和newStartVnode 或者 oldEndVnode和newEndVnode 满足sameVnode，直接将该VNode节点进行patchVnode即可，不需再遍历就完成了一次循环，如下图

<img src="https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122154819062.png" alt="image-20201122154819062" style="zoom: 45%;" />

如果oldStartVnode与newEndVnode满足sameVnode，说明oldStartVnode已经跑到了oldEndVnode后面去了，进行patchVnode的同时还需要将真实DOM节点移动到oldEndVnode的后面，因为在新数组中oldStartVnode已经在队尾了

> 注意：新老两组VNode节点是不移动的，只是索引移动
>
> 这里操作的是真实DOM，一边比较一边更新真实DOM，但会不会引起页面多次触发重绘重排呢，答案是不会的，这里虽然是分开多次更新DOM操作，但在Vue中都是同一个"Tick"时间触发更新的，而浏览器对短时间内触发的DOMDOM有优化策略，大多数浏览器都会通过队列化修改并批量执行来优化重排过程，浏览器会将修改操作放入到队列里，直到过了一段时间或者操作达到了一个阈值，才清空队列，触发重绘重排

<img src="https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122160056951.png" alt="image-20201122160056951" style="zoom:47%;" />

如果oldEndVnode与newStartVnode满足sameVnode，说明oldEndVnode跑到了oldStartVnode的前面，进行patchVnode的同时要将oldEndVnode对应DOM移动到oldStartVnode对应DOM的前面

![image-20201122155034308](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122155034308.png)

如果以上情况均不符合，则在old VNode中找与newStartVnode满足sameVnode的vnodeToMove，若存在执行patchVnode，同时将vnodeToMove对应DOM移动到oldStartVnode对应的DOM的前面

<img src="https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122155213378.png" alt="image-20201122155213378" style="zoom:47%;" />

当然也有可能newStartVnode在old VNode节点中找不到一致的key，或者是即便key相同却不是 sameVnode，这个时候会调用createElm创建一个新的DOM节点

![image-20201122155335770](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122155335770.png)

至此循环结束，但是我们还需要处理剩下的节点

当结束时oldStartIdx > oldEndIdx，这个时候旧的VNode节点已经遍历完了，但是新的节点还没有，说明了新的VNode节点实际上比老的VNode节点多，需要将剩下的VNode对应的DOM插入到真实DOM 中，此时调用addVnodes（批量调用createElm接口）

<img src="https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122155529148.png" alt="image-20201122155529148" style="zoom:47%;" />

但是，当结束时newStartIdx > newEndIdx时，说明新的VNode节点已经遍历完了，但是老的节点还有剩余，需要从文档中删 的节点删除

![image-20201122155622786](https://raw.githubusercontent.com/byCHENJIAJIE/image-hosting/master/typora/image-20201122155622786.png)

```js
// src/core/vdom/patch.js
export function createPatchFunction (backend) {
  // ...
  
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    // 4个游标
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    // 对应的4个vnode
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]

    // 后续查找需要的变量
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    // 注意结束条件，开头和结束游标不能重合，即开始索引不能大于结束索引
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 这个节点在 4种最可能出现的情况没有找到相同时，已经遍历处理过了
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 这个节点在 4种最可能出现的情况没有找到相同时，已经遍历处理过了
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 老开始和新开始相同
        // 两者patch
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        // 索引向后移动一位
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 老结束和新结束相同
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 老开始和新结束相同
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        // 除了patch还要把oldStartVnode移动到oldCh队尾
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 老结束和新开始相同
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        // 移动到队首
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 上面4种最可能出现的情况没有找到相同，则循环查找
        // 先从新的头部拿一个，去老的里面找相同
        // 这里有两种方式寻找
        // 一种是通过遍历oldCh的oldStartIdx到oldEndIdx
        // 在2.4.2上加上了一个用节点的key做的map来匹配
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        // 没找到，创建
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          // 找到，patch
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            // patch同时移动到队首
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }

    // 有一个数组全部处理完成，另一个必定未处理完成
    if (oldStartIdx > oldEndIdx) {
      // 老数组结束，证明新增了节点，批量创建
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // 新数组结束，证明删除了节点，批量删除
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
}

// 判断两个节点是相同节点
// 核心的两个必要条件：key和tag
function sameVnode (a, b) {
  return (
    // key值
    a.key === b.key && (
      (
        // 标签名
        a.tag === b.tag &&
        // 是否为注释节点
        a.isComment === b.isComment &&
        // 是否都定义了data，data包含一些具体信息，例如onclick , style
        isDef(a.data) === isDef(b.data) &&
        // 当标签是<input>的时候，type必须相同
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

##### 补充key属性说明

**预期**：`number | string | boolean (2.4.2 新增) | symbol (2.5.12 新增)`

`key` 的特殊 attribute 主要用在 Vue 的虚拟 DOM 算法，在新旧 nodes 对比时辨识 VNodes。如果不使用 key，Vue 会使用一种最大限度减少动态元素并且尽可能的尝试就地修改/复用相同类型元素的算法。而使用 key 时，它会基于 key 的变化重新排列元素顺序，并且会移除 key 不存在的元素

有相同父元素的子元素必须有**独特的 key**。重复的 key 会造成渲染错误

最常见的用例是结合 `v-for`，当 Vue 正在更新使用 `v-for` 渲染的元素列表时，它默认使用“就地更新”的策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序，而是就地更新每个元素，并且确保它们在每个索引位置正确渲染。这个类似 Vue 1.x 的 `track-by="$index"`

这个默认的模式是高效的，但是**只适用于不依赖子组件状态或临时 DOM 状态 (例如：表单输入值) 的列表渲染输出**

为了给 Vue 一个提示，以便它能跟踪每个节点的身份，从而重用和重新排序现有元素，你需要为每项提供一个唯一 `key` attribute：

```html
<ul>
  <li v-for="item in items" :key="item.id">...</li>
</ul>
```

建议尽可能在使用 `v-for` 时提供 `key` attribute，除非遍历输出的 DOM 内容非常简单，或者是刻意依赖默认行为以获取性能上的提升

在下面的代码中切换 `loginType` 将不会清除用户已经输入的内容。因为两个模板使用了相同的元素，`<input>` 不会被替换掉——仅仅是替换了它的 `placeholder`

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address">
</template>
```

也可以加上key表达“这两个元素是完全独立的，不要复用它们”

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username" key="username-input">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address" key="email-input">
</template>
```

现在，每次切换时，输入框都将被重新渲染，用于强制替换元素/组件而不是重复使用它。当你遇到如下场景时它可能会很有用：

- 完整地触发组件的生命周期钩子
- 触发过渡

例如：

```html
<transition>
  <span :key="text">{{ text }}</span>
</transition>
```

当 `text` 发生改变时，`<span>` 总是会被替换而不是被修改，因此会触发过渡



