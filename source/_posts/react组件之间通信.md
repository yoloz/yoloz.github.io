---
title: react组件之间通信
comments: false
toc: false
date: 2019-02-20 16:08:39
categories: web
tags:
---

在React组件中，props是父组件与子组件的唯一通信方式，但是在某些情况下我们需要在props之外强制修改子组件或DOM元素，这种情况下React提供了Refs解决。
> 如果能使用props实现，应该尽量避免使用refs实现。

## Refs三种方式
* \*字符串模式 ：废弃不建议使用
* \*回调函数
* \*React.createRef()：React16.3提供

<!--more-->

## 字符串模式
``` js
class List extends React.Component {
  constructor(props, context) {
    super(props, context);
  }
  componentDidMount(){
    this.refs.inputEl.focus();
  }
  render() {
    const { list } = this.props;
    return (
        <input ref="inputEl"/> 
    );
  }
}
```
### 存在的问题
* \*针对静态类型检测不支持
* \*对复杂用例难以实现：需要向父组件暴露dom；单个实例绑定多个dom
* \*绑定到的实例，是执行render方法的实例，结果会让人很意外，例如：
``` js
class Child extends React.Component {
  render() {
    const { renderer } = this.props;
    return <div>{renderer(1)}</div>;
  }
}
class App extends React.Component {
  render() {
    return (
      <div className="App">
        <Child renderer={index => <div ref="test">{index}</div>} />
      </div>
    );
  }
}
```
> 上面这种情况，会导致test绑定的实例是Child上面，并不是App上。

## 回调函数模式
相比字符串模式更加灵活，也避免了诸多问题。
* \*可以优雅在组件销毁时回收变量, ref中的回调函数会在对应的普通组件componentDidMount，ComponentDidUpdate之前; 或者componentWillUnmount之后执行，componentWillUnmount之后执行时，callback接收到的参数是null。
* \*很好的支持静态类型检测。
* \*针对数组遍历时可以直接转换为对应的数组，看下面的例子：

``` js
class List extends React.Component {
  constructor(props, context) {
    super(props, context);
  }
  _ref = el => {
    if (el) {
      if (!this.els) {
        this.els = [];
      }
      this.els.push(el);
    } else {
      this.els = [];
    }
  };
  render() {
    const { list } = this.props;
    return (
      <ul>
        {list.map((item, index) => {
          return (
            <li ref={this._ref} key={index}>
              {item}
            </li>
          );
        })}
      </ul>
    );
  }
}
class App extends React.Component {
  state = {
    value: "",
    list: []
  };
  onChange = ({ target: { value } }) => {
    this.setState({ value });
  };
  add = () => {
    const { list, value } = this.state;
    list.push(value);
    this.setState({
      value: "",
      list
    });
  };
  render() {
    const { value, list } = this.state;
    return (
      <div className="App">
        <input value={value} onChange={this.onChange} />
        <button onClick={this.add}>add</button>
        <List list={list} />
      </div>
    );
  }
}
```
* \*可以将父组件的ref传入孙组件，虽然不建议这么使用（破坏组件封装）。
``` js
function Input(props) {
  return (
    <div>
      <input ref={props.inputRef} />
    </div>
  );
}

class InputBox extends React.Component {
 _ref = (el)=>{
     this.inputElement = el
 }
  render() {
    return (
      <Input
        inputRef={this._ref}
      />
    );
  }
}
```
### 同样存在弊端
通常为了绑定一个组件（元素）实例到当前实例上需要写一个函数，代码结构上看起来很冗余，为了一个变量，使用一个函数去绑定，每一个绑定组件（元素）都需要一个方法处理，大材小用。因此React提供了更简单有效的解决方案`React.createRef()`。

## React.createRef()
使用React.createRef()创建refs，通过ref属性来获得React元素。当构造组件时，refs通常被赋值给实例的一个属性，这样你可以在组件中任意一处使用它们.
``` js
class Test extends React.Component{
    myRef = React.createRef();
    componentDidMount(){
      // 访问ref
      const dom = this.myRef.current
    }
    render(){
        return (
            <div ref={this.myRef}/>
        )
    }
}
```
ref的值取决于节点的类型:
* \*当ref属性被用于一个普通的HTML元素时，React.createRef()将接收底层DOM元素作为它的current属性以创建ref 。
* \*当ref属性被用于一个自定义类组件时，ref对象将接收该组件已挂载的实例作为它的current 。
* \*你不能在函数式组件上使用ref属性，因为它们没有实例。

### refs传递
将父组件ref作为一个props传入，在子组件显示调用
``` js
class Sub extends Component{
    render(){
        const {forwardRef} = this.props;
        return <div ref={forwardRef}/>
    }
}
class Sup extends Component{
    subRef = React.createRef();
    render(){
        return <Sub forwardRef={this.subRef}/>
    }
}
```
## React.forwardRef()
React.forwardRef方式，对于使用组件者来说，ref是透明的，不需要额外定一个props传入，直接传递到了下级组件，作为高阶组件封装时，这样做更加友好。
``` js
class Sub extends Component{
    render(){
        const {forwardRef} = this.props;
        return <div ref={forwardRef}/>
    }
}

function forwardRef(props, ref){
    return <Sup {...props} forwardRef={ref}/>
}
// 为了devtool中展示有意义的组件名称
forwardRef.displayName=`forwardRef-${Component.displayName||Component.name}`

const XSub = React.forwardRef(forwardRef);

class Sup extends Component{
    _ref=(el)=>{this.subEl =el};
    render(){
        return <XSub ref={this._ref}/>
    }
}
```

## 总结
Refs字符串模式已经废弃，React不建议使用并且会提示警告，开发中推荐使用React.forwardRef方式，简单优雅，回调函数模式应用在复杂场景中。
上述内容来自[浅谈 React Refs](https://imweb.io/topic/5b6136a06025939b125f45ff)

## onRef
如果Parent调用Child下的Child(或更多)，Child及它的Child同样添加onRef，然后Parent里面`let tableData = this.child.child.child.state.data;`这样取值。如下:
``` js
import React, {Component} from 'react';
export default class Parent extends Component {
    render() {
        return(
            <div>
                <Child onRef={this.onRef} />
                <button onClick={this.click} >click</button>
            </div>
        )
    }
    onRef = (ref) => {
        this.child = ref
    }
    click = (e) => {
        this.child.myName()
    }
}
class Child extends Component {
    componentDidMount(){
        this.props.onRef(this)
    }
    myName = () => alert("组件之间通信成功");
    render() {
        return ();
    }
}
```



