---
title: react-loadable代码分割
comments: false
toc: false
date: 2018-11-29 14:54:43
categories: 
tags:
---

* 安装依赖 `yarn add react-loadable`
* 根据[react-loadable文档](https://github.com/jamiebuilds/react-loadable/blob/master/README.md)提示，我们需要提供一个载入新页面时的Loading组件，同时对加载和超时状态进行区别提示：

``` js
import React from 'react';
import { Icon } from 'antd';

const Loading = ({ pastDelay, timedOut, error }) => {
  if (pastDelay) {
    return <div><Icon type="loading" /></div>;
  } else if (timedOut) {
    return <div>Taking a long time...</div>;
  } else if (error) {
    return <div>Error!</div>;
  }
  return null;
};
```


* 分割各组件：

``` js
import React from 'react';
import Loadable from 'react-loadable';
import Loading from "./loading";

const Home = Loadable({
  loader: () => import('../Home'),
  loading: Loading,
  timeout: 10000
});
const EditArticle = Loadable({
  loader: () => import('../EditArticle'),
  loading: Loading,
  timeout: 10000
});
...
export {Home, EditArticle,...}
```

* 更改入口文件

``` js
import React, { Component } from 'react';
import { Route, Switch } from 'react-router-dom'
import {Home, EditArticle,...} from './loadable'

class App extends Component {  
  constructor(props) {
  }
  render() {
    return (
        <Router>
        <Switch>
        <Route exact path='/home' component={Home}/>
        <Route path='/editarticle' component={EditArticle} />
        </Switch>
        </Router>
    )
  }
}
export default App;
```
