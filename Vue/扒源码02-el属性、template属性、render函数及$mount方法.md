### el属性、template属性及render函数的区别，$mount 和el的区别

```js
// src/platforms/web/entry-runtime-with-compiler.js
// 扩展默认$mount方法：处理template或el选项

// 引入默认$mount方法，并扩展它
// 编译template
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 查询得到真实dom
  el = el && query(el)

  // 处理render或者template或者el三个选项
  // 编译并得到渲染函数
  const options = this.$options
  // resolve template/el and convert to render function
  // render > template > el
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }

    // 获取到模板之后，编译之
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      // 获取渲染函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  // 执行挂载
  return mount.call(this, el, hydrating)
}
```

共同点都是围绕初始化Vue实例时指定模板和挂载元素两个作用

**template属性**（只指定模板）：

- 当为html字符串时，直接使用html字符串作为模板
- 当为#开头的字符串时，使用匹配到的ID元素innerHTML作为模板
- 当为元素节目时，直接使用该元素innerHTML作为模板

无论上面哪种情况，template属性最后都要手动$mount挂载元素

**el属性**（指定模板和挂载元素）：直接使用el元素的outerHTML作为模板并已该ID元素作为挂载元素（Firefox不支持outerHTML，源码通过创建一个空节点，然后appendChild el元素，再读取原空节点的innerHTML获取）

el等同于template+$mount挂载元素的效果，不同在于el为自动调用$mount挂载元素，而template为手动调用$mount，在项目中可用于延时挂载（例如在挂载之前要进行一些其他操作、判断等）

```js
// src/core/instance/index.js
// Vue构造函数
function Vue (options) {
  // 初始化
  this._init(options)
}

// src/core/instance/init.js
export function initMixin (Vue: Class<Component>) {
	// 对Vue扩展，实现_init实例方法
  Vue.prototype._init = function (options?: Object) {
    // ...
    
    // 选项合并
    // ...
    
    // 初始化核心代码
    // ...

    // 调用$mount
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

**render函数**（创建模板）：优先级最高，通过传入的createElement方法直接创建模板HTML内容，然后使用el属性或$mount指定挂载元素

template属性和el属性最后都是通过编译器Compiler编译为render函数供Vue初始化使用，所以使用template属性和el属性作为指定模板功能时要使用带有编译器Compiler版本的Vue，render函数+el属性作为指定挂载元素作用时只需要Runtime版本Vue

优先级：render > template > el