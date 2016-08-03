# react-update

[![build status](https://img.shields.io/travis/jianghai/react-update.svg)](https://travis-ci.org/jianghai/react-update)

Make setState easily and immutably.

## 解决什么问题？

1、我们需要不可变数据，利用 shouldComponentUpdate 做性能优化

2、很多 state 结构是嵌套的，如 Tree 组件、父子组件通信等场景

3、实际开发中，大量的交互只是对 state 的处理，可以抽象成对数据的赋值、添加、删除等操作，减少了大量的重复代码


## Installation

```sh
npm i --save react-update
```

## Usage

#### 单个组件

```javascript
import update from 'react-update'

class App extends Component {
  
  constructor() {
    super()
    this.update = update.bind(this)
  }
  
  handleAdd(text) {
    this.update('push', 'list', { text })
  }
}
```

#### 组件间通信

父组件－中间组件－子组件为例，子组件的数据随父级变化而变化，因此子组件“改变”数据后需要通知父组件。

但请注意，子组件通知跨级的父组件并不需要一级一级往上回调，而是父组件的回调已预先传给子组件，中间组件只是负责父组件的回调转发给子组件即可。

此外，中间组件的转发并非毫无意义，如果抛开父组件，中间组件定义好的 props 本身就已明确了输入和输出，又可以被其它场景下复用了。

```javascript
import update from 'react-update'

class Parent extends Component {
  
  constructor() {
    super()
    this.update = update.bind(this)
  }
  
  handleChange(...args) {
    this.update(args)
  }

  render() {
    return <Child onChange={::this.handleChange} />
  }
}

class Middle extends Component {
  render() {
    return <Child {...this.props} />
  }
}

class Child extends Component {
  
  handleChange(...args) {
    this.props.onChange(args)
  }

  render() {
    return (
      <div>
        <input type="text" onChange={e => this.handleChange('set', 'name', e.target.value)}>
        <button onClick={() => this.handleChange('push', 'list', {})}>Add</button>
      </div>
    )
  }
}
```

## API

```javascript

// state
{
  x: 1,
  a: {
    b: 1
  },
  list: [0]
}

// 基本的 set, push, splice
update('set', 'x', 0)
update('set', 'a.b', 0)
update('set', ['a', 'b'], 0)
update('push', 'list', 1)
update('splice', 'list', 0) // 0 is the index which would be removed
update('set', 'list[0]', 'hello')

// 多条操作
// 相同类型
update('set', {
  x: 2,
  'a.b': 2
})
// 不同类型
update(['set', 'x', 0], ['push', 'list', 1])

// 返回值，当一些操作依赖更新后的结果时
update('set', 'x', 0) // => 0
update('set', {
  'a.b': 0, 
  'a.c': 0
}) // => {b: 0, c: 0}
update(['set', 'x', 0], ['push', 'list', 1]) // => {x: 0, list: [0, 1]}

// 多次调用不会造成多次渲染，后面的更新也不会覆盖前面的更新
update('set', 'a.b', 0)
update('set', 'a.c', 0)
// => {a: {b: 0, c: 0}}
```

## 开发者调试功能

非 production 模式下，调用 update 后，控制台输出 update 所属组件更新后的 state，方便定位问题


## shouldComponentUpdate 优化

update 的处理过程是不可变的，结合 shouldComponentUpdate 优化才能发挥其核心价值，为了省事，无需每个组件单独添加 shouldComponentUpdate 方法，全局定义一下即可:

```javascript
// ShadowEqual except function props.
function isEqual(source, target) {
  if (!source) return true
  return Object.keys(source).every(key => {
    let isEqual = true
    const prop = source[key]
    if (typeof prop !== 'function' && target[key] !== source[key]) {
      isEqual = false
    }
    return isEqual
  })
}

// Global optimizing for immutable data.
const prototype = React.Component.prototype
if (!prototype.shouldComponentUpdate) {
  prototype.shouldComponentUpdate = function(nextProps, nextState) {
    if (nextProps.children) return true
    return !(isEqual(nextProps, this.props) && isEqual(nextState, this.state))
  }
}
```

组件写法上也要注意：

1、使用 class 代替 React.createClass

2、除函数类型外，不要重写创建对象，否则无法保证相等

例如

```javascript
render() {
  return <div style={{color: '#999'}}></div>
}
```

需要改成

```javascript
constructor() {
  this.style = {
    color: '#999'
  }
}
render() {
  return <div style={this.style}></div>
}
```