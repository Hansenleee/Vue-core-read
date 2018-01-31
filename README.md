# vue-core-read
记录vue源码总结

## 核心功能

vue的核心功能主要依赖于三个类，Observer、Dep、Watch，先简单介绍基础流程
* vue初始化时（vm._init）会将props，data，computed，watcher通过Observer初始化set和get，创建依赖dep
* 解析template模板，将指令通过Watcher实例，触发上一条的get，将watcher放入对应的deps中
* 每当this.** = 改变时，通过set方法，调用收集到的依赖触发dep.notify方法去发布
至此，vue的订阅发布和依赖收集模式基本完结，接下去分析具体实现方法

### Observer
首先vue初始实例时`Vue.prototype._init`会初始化props、computed、methods...
```javascript
<目录：src/core/instance/init.js>
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```
我们着重介绍下`initState(vm)`函数，initState函数中会对props，data，computed进行proxy代理，具体操作是利用
Object.defineProperty简单的将this.data.num代理到this.num上。然后利用observer方法对各个属性进行初始化。observer函数如下
```javascript
<目录：src/core/observer/index.js>
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
其中主要是判断value是否已经处理过（Observer构造器中会在value上设置__ob__属性）如果处理过直接返回，没处理返回进行Observer类实例后的值。
Observer类的代码如下所示

Observer类源码
```javascript
<目录：src/core/observer/index.js>
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```
Observer构造函数中会对value进行递归，最终都会调用this.walk方法，walk方法接着遍历调用defineReactive方法
defineReactive如下所示
```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```
该方法中主要是通过`Object.defineProperty`对key-value设置get和set。其中涉及到的Dep稍后讲解。
* get方法：将当前正在计算的属性（Dep.target）加入到它的deps依赖中。Dep.target是个Watcher，也会在稍后说明，这里可以理解为：通过get方法来添加依赖
* set方法：上面通过get来收集依赖,这里通过set在当前属性改变时去实现发布功能。dep.notify()会触发收集的依赖的update方法

### Dep
Dep类的代码并不多，它只是个依赖的容器，不进行实际的处理，仅仅只是提供订阅发布功能。Dep类代码如下所示
```javascript
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```
这里的逻辑并不多，主要是维护`this.subs`这个依赖的数组，addSub是添加依赖，removeSub是移除依赖，notify是当属性改变时通知依赖调用update方法进行更新。唯一比较难理解的是depend函数。那么问题来了，Dep.target究竟是个什么东西呢？它是正在计算的Watcher（vue中同时只会有一个Watcher正在计算）所以depend是把当前计算的watcher加入到deps依赖中。下面是对Dep.target进行复制和移除的方法,无需多言
```javascript
export function pushTarget (_target: Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

### Watcher
Watcher实现的方法有些复杂，我们一步步来看代码。
```javascript
```

## 工具函数

### parsePath函数
解析template中的vue实例路径---例如`<div>{{test.count}}</div>`会将test.count解析成
`['test', 'count']`数组，外部调用返回的函数时通过传入的参数obj（vue实例）时，循环数组依次
取值`obj = vm.test`,`obj = vm.test.count`

```javascript
const bailRE = /[^\w.$]/
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```
