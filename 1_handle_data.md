```javascript
new Vue({
  el: '#app',
  data: {
    message: 'Hello World!',
  },
})
```

这是我在使用Vue时写下的第一段代码，我会好奇，在运行这段代码时，其背后Vue究竟完成了哪些操作呢？阅读并理解源码，让我们看到隐藏在框架后的那一片片齿轮，在需要性能优化，或是遇到疑难解决错误时，也许能更轻松地解决问题。

刚开始学习编程时，一般会寻找程序运行的入口。在src/core/instance/index.js中，可以看到对Vue的定义，也是我们所寻找的入口。

约定在先，文章中只摘录部分源码，建议在阅读之后，可以按照文章中的思路顺序看一遍源代码，所有的源码基于Vue@2.16.10版本。在引用的代码之前，都会用注释标记这段代码来自于哪个文件。

```javascript
// file: src/core/instance/index.js
function Vue(options) {
  // ..
  this._init(options);
}

// ..
initMixin(Vue);
stateMixin(Vue);
```

实际上HelloWorld项目中调用Vue构造函数时，最终会调用initState()

```javascript
// file: src/core/instance/init.js
import { initState } from './state'

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    // ..
    initState(vm)
    // ..
  }
}
```

```javascript
// file: src/core/instance/state.js
export function initState(vm: Component) {
  // ..
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  // 对于data的处理
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

### 处理Data数据

在initState中，可以看到对开发过程中常接触到的`props/data/computed`的处理函数，我们来看看对于data的处理过程——initData()。

```javascript
// file: src/core/instance/state.js
function initData(vm: Component) {
  // ..
  observe(data); // 最后是使用observe函数对data进行了处理
}
```

来查看observe函数中对于value，也就是data的处理

```javascript
// file: src/core/observer/index.js
export function observe(value: any, asRootData: ?boolean): Observer | void {
  // ..
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 参数value将是定义的data对象
    ob = new Observer(value)
  }
  // ..
  return ob
}
```

其中最主要的是创建了一个Observer对象，来看看Observer对象的构造流程

```javascript
// file: src/core/observer/index.js
export class Observer {
  // ..
  constructor (value: any) {
    this.value = value
    this.dep = new Dep() // 之后会使用到
    // ..
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // ..
      this.observeArray(value)
    } else {
      // 因为目前定义的data是一个object，所以会执行walk函数
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

在Observer构造函数中，因为我们在HelloWorld项目中传入的是一个object，最终将调用Observer了walk函数，即对于data中的每个property调用了defineReactive函数进行处理。

### 调用defineReactive函数对property进行改造

来看看defineReactive所进行的处理。其中用来到[property descriptor](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)，如果不了解的话，建议优先弄清楚再继续阅读。

```javascript
// file: src/core/observer/index.js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  // 获取descriptor
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // 如果descriptor定明确说明该property不能修改，那么不进行处理
  if (property && property.configurable === false) {
    return
  }

  // 获取已经定义的get和set函数，应该都是undefined
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // ..
  // 重新定义property
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      // ..
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      // ..
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        // 将执行到这里
        val = newVal
      }
      // ..
    }
  })
}
```

在defineReactive中，重新将data中的message定义为accessor descriptor。当我们使用data.message时，调用reactiveGetter函数，最终返回的是val，当我们为data.message重新赋值时，调用reactiveSetter函数，实际上是为val重新赋值。

### 结论

对于data的处理可以简单地理解为，重新定义每个property，将其从value descriptor改造成accessor descriptor，至于为什么要这么做，在之后的`computed/watcher`的定义中会找到使用的地方。