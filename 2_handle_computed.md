到此为止，基本上data相关的处理流程就已经完成了，在这里有被所跳过的props的定义，现在来看一下。

```javascript
// src/core/instance/state.js
function initProps(vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      // ..
    } else {
      defineReactive(props, key, value)
    }
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  // ..
}
```

#### 结论：
基本上对于props的处理，就是循环调用defineReactive函数进行处理。

### 处理Computed

```javascript
new Vue({
  el: '#app',
  data() {
    return {
      d1: 'd1',
    };
  },
  computed: {
    c1() {
      return d1;
    },
  },
});
```

关于computed被如何处理，也用上面一个简单的代码片段来说明。
让我们来看initComputed，也是针对于每个computed中的属性进行处理，其中比较关键的是new Watcher，在Watcher中主要执行的pushTarget getter() popTarget 这三个函数。我们来针对这段代码进行处理

当执行pushTarget的时候，会将d1所对应的Watcher放在Dep.target的位置

computed是如何注册在vm上，也是使用defineProperty，使用accessor descriptor的形式注册在vm上，这里的getter使用createComputedGetter来实现，在createComputedGetter中使用了watcher.depend来完成对于其他内容的dep

我们来看看initComputed函数是如何对于computed数据进行处理的

```javascript
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

来查看在Watcher的构造函数中完成了哪些工作

```javascript
class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }
}
```

(可以在这里做一个stack中状态的gif动图

```javascript
// 考虑这个vue代码，其执行逻辑大致是怎样的呢
new Vue({
  el: '#app',
  data() {
    return {
      d1: 'd1',
    };
  },
  computed: {
    c1() {
      return d1;
    },
    c2() {
      return c1;
    },
  },
});
```

所给出的总结和思考是，在看源码的过程中，要自己完成对于源码的注释。
发现自己暂时看不懂的逻辑时，也可以先注释标记一下，在之后的过程中，也许能搞明白。