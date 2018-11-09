# Vue 原理
## 模板编译器
Vue实现了一个完整的类HTML模板的编译器，所谓编译时把一种形式的语法转换成另外一种形式的语法。这里是把Vue的模板语法转换render函数。

编译分成三个阶段，parse解析、optimize优化和generate代码生成。

### 解析

解析过程涉及到一些编译原理知识，这里不详细展开。通过对字符的移进、归约，以及正则匹配等方式将模板字符串进行解析，得到指令、class、style等结构化的数据对象，这个对象我们称之为构建中间产物。

源模板结构：
```
<ul>
  <li v-for="(obj, key, index) in list">
    <span>文本</span>
  </li>
</ul>
```

模板解析后的中间产物结构如下：

```
{
  type: 1,
  tag: 'ul',
  parent: null,
  attrsList: [],
  children: [
    {
      type: 1,
      tag: 'li',
      parent: ul,
      attrsList: [],
      for: 'list',
      alias: 'obj',
      iterator1: 'key',
      iterator2: 'index'
      children: [
        {
          type: 1,
          tag: 'span',
          parent: li,
          attrsList: [],
          children: [
            {
              type: 2,
              tag: '',
              parent: span,
              attrsList: [],
              text: '文本'
            }
          ]
        }
      ]
    }
  ]
}
```

### 优化

优化是针对前一步的中间产物进行一些标记，以方便在运行阶段获取更高的性能，这里 Vue 会将一些不会发生变化的节点进行 static  标记，方便在后续 Diff 过程中直接跳过。

优化后的中间产物结构：

```
{
  type: 2,
  tag: '',
  parent: span,
  attrsList: [],
  text: '文本',
  static: true
}
```

### 代码生成

代码生成是将中间产物转成render函数的字符串。

Vue.js 的render函数如下：

```
with(this) {
  return _c(
    'div',
    {
      class: c
    },
    _l(sz, function(item) {
      return _c('span', [_v('hello world')])
    })
  );
}
```

## Diff算法

Diff 算法是通过同层的树节点进行比较而非对树进行逐层搜索遍历的方式，所以时间复杂度只有 O(n)， 是一种相对高效的算法，如下图：
![diff](/vue/diff.png)

相同颜色的方块中的节点会进行比对，比对得到「差异」后将这些「差异」更新到 DOM 上。
## 响应式
Vue内部的defineReactiv方法通过Object.defineProperty来实现对对象的”响应式“化。经过defineReactive处理以后，obj的key属性在读的时候回触发reactiveGetter方法，而在写的时候会触发reactiveSetter方法。
```
/*
 * obj: 目标对象
 * prop: 需要操作的目标对象的属性名 descriptor: 描述符
 * return value 传入对象 
*/
Object.defineProperty(obj, prop, descriptor)
```
针对数组数据，Vue进行了特殊的处理，替换了数组对象的原先，劫持了一些数组方法入pop、push、slice等，对这些数组操作后的值遍历进行defineReactive的处理，以保证这些数组方法的返回值依旧是可相应的数据。

在Vue的构造函数中，响应式系统处理的是data参数，代码简化如下：
```
class Vue{
	constructor(options){
		this._data = options.data;
      observer(this._data);
}
}
```

## 依赖收集
依赖收集使得Vue的响应系统变得高效。假设我们有一公共的数据对象声明data.js中
```
module.exports = {
foo: 'bar'
}
```

同事又有两个Vue组件（a.vue和b.vue）引用了这个对象中的数据。
```
import data from './data';
export default {
	data() {
		return data;
},
template: `<div>{foo}</div>`
}
```

```
/* b.vue */
import data from './data';
export default {
  data() {
    return data;
  },
  template: `<span>{foo}</span>`
}
```

这个时候我们在a组件中修改data中的数据： this.foo = ‘hello world!’;这个时候Vue应该通知a组件和b两个vm实例进行视图更新，但是Vue如何知道需要通知他们呢？依赖收集的过程就是让响应式数据data知道哪些地方依赖了它，在其变化的时候通知他的依赖方进行视图的更新。

## 订阅者
订阅者Vue中被称为Dep，他的主要作用是用来存放观察者对象，并可以通知其依赖观察者数据发生了变化。
```
class Dep {
	constructor () {
	this.subs = []; 
}

addSub(sub){
this.subs.push(sub);
}

notify() {
	this.subs.forEach(sub = >{
sub.update();
})
}
}
```

## 观察者
观察者提供了一个update方法，用于接受数据变更的时机。

```
class Watcher{
constructor() {
Dep.target = this;
}

update() {
	console.log('视图更新啦')
}
}
```

在对象被 读 的 时候，会触发reactiveGetter函数把当前的Watcher对象（存放在Dep.target中）收集到Dep中。