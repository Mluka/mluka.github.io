---
layout:     post
title:      "React.PureComponent全解析"
subtitle:   "Component，你可不够纯哦"
date:       2019-02-01
author:     "MeloGuo"
header-img: "img/post-bg-react.png"
tags:
    - JavaScript
    - React
---

# React.PureComponent
在 v15.3 中增加了一个新 API，从名字可以看出 PureComponent 应该比 Component 更纯。文档中解释，它们的不同之处在于 React.Component 没有实现 shouldComponentUpdate()，但是 React.PureComponent实现了它。采用对属性和状态用浅比较的方式进行更新。

## 如何浅比较？
浅比较中会比较 Object.keys(state || props) 的长度是否一致，每一个 key 是否两者都有，并且是否是一个引用，也就是只比较了第一层的值，确实很浅，所以深层的嵌套数据是对比不出来的。

```javascript
function shallowEqual (prevVal, newVal) {
  const prevValKeys = Object.keys(prevVal)
  const newValKeys = Object.keys(newVal)

  if (prevValKeys.length !== newValKeys.length) { return false }

  return prevValKeys.every(prevValKey => newValKeys.indexOf(prevValKey) > -1 && prevVal[prevValKey] === newVal[prevValKey])
}
```

## PureComponent 是如何使用 shallowEqual 的？
先看一个真正 PureComponent 的例子：

```javascript
class App extends React.PureComponent {
  state = {
    items: [1, 2, 3]
  }

  handleClick = () => {
    const { items } = this.state
    items.pop()
    this.setState({ items })
  }

  render () {
    return (
      <div>
        <ul>
          {this.state.items.map((item, index) => <li key={index}>{item}</li>)}
        </ul>
        <button onClick={this.handleClick}>删除</button>
      </div>
    )
  }
}
```
在这个组件中，你会发现点击删除按钮是无效的！原因就是新的`state.item`和旧的`state.item`指向同一个引用，所以被 PureComponent 拒绝了渲染。这也正是 PureComponent 的特性。在 Component 中使用 shallowEqual 这个函数模拟一下便也有这样的效果。

```javascript
class App extends React.Component {
  state = {
    items: [1, 2, 3]
  }

  // 模拟 PureComponent
  shouldComponentUpdate (nextProps, nextState) {
    return !shallowEqual(this.props, nextProps) || !shallowEqual(this.state, nextState)
  }

  handleClick = () => {
    const { items } = this.state
    items.pop()
    this.setState({ items })
  }

  render () {
    return (
      <div>
        <ul>
          {this.state.items.map((item, index) => <li key={index}>{item}</li>)}
        </ul>
        <button onClick={this.handleClick}>delete</button>
      </div>
    )
  }
}
```
所以在 PureComponent 中更新引用类型的数据时不要使用同一引用，而应该使用一个新的引用，如下：

```javascript
handleClick = () => {
    const { items } = this.state
    items.pop()
    // 使用一个新的引用
    this.setState({ items: [...items] })
  }
```
这样便不会被判定为 state 未改变。但是对于不变的数据却应该使用同一个引用，不要每次 render 都让引用变化引发不必要的重渲染：
```javascript
update () {
    console.log('update')
}
render () {
    // 每次渲染都会传入一个新的引用的函数
    return <div onClick={this.update.bind(this)} />
}
```
解决方法便是每次传入相同引用的函数
```javascript
update = () => {
    console.log('update')
}
render () {
    return <div onClick={this.update} />
}
```
在处理对象时也有相似的问题，例如为属性指定默认值：

```javascript
render () {
    // 在 JS 中 {} 不等于 {}
    return <div data={this.state.data || {}} />
}
```
这种情况每次传入的默认值都会被判定为新值从而进行渲染，所以应该使用一个变量固定引用。

```javascript
defalutData = {}

render () {
    return <div data={this.state.data || this.defalutData} />
}
```
## 总结
PureComponent真正起作用的，只是在一些纯展示组件上，复杂组件用了也没关系，但是在 shadowEqual 那里便会被拦截，不过记得 props 和 state 不能使用同一个引用哦。


#### 著作权声明

本文作者 [郭梓梁](https://www.zhihu.com/people/mluka/activities)，首次发布于 [MeloGuo Blog](http://meloguo.com)，转载请保留以上链接
