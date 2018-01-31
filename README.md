# vue-core-read
记录vue源码总结

## 核心功能-数据绑定、依赖收集、订阅发布模式

vue的核心功能主要依赖于三个类，Observer、Dep、Watch，先简单介绍基础流程
* vue初始化时（vm._init）会将props，data，computed，watcher通过Observer初始化set和get，创建依赖dep
* 解析template模板，将指令通过Watcher实例，触发上一条的get，将watcher放入对应的deps中
* 每当this.** = 改变时，通过set方法，调用收集到的依赖dep.notify方法去发布
至此，vue的订阅发布和依赖收集模式基本完结，接下去分析具体实现方法

### Observer

### Dep

### Watcher

## 工具函数

### parsePath函数
解析template中的vue实例路径---例如`<div>{{test.count}}</div>`会将test.count解析成
`['test', 'count']`数组，外部调用返回的函数时通过传入的参数obj（vue实例）时，循环数组依次
取值`obj = vm.test`,`obj = vm.test.count`

```
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
