---
title: Vue cdn加速配置
date: 2021-05-07 11:34
tags: vue
categories: 
---

<!--more-->

## 引入CDN

```html
    <script src="https://cdn.bootcdn.net/ajax/libs/vue/2.6.10/vue.min.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/vue-router/3.0.6/vue-router.min.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/vuex/3.1.0/vuex.min.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/axios/0.18.1/axios.min.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/element-ui/2.13.0/index.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/js-cookie/2.2.0/js.cookie.min.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/nprogress/0.2.0/nprogress.js"></script>
    <link href="https://cdn.bootcdn.net/ajax/libs/element-ui/2.13.0/theme-chalk/index.css" rel="stylesheet">
```

## 配置vue.config.js

在module.exports中引入的cdn不需要打包

```javascript
configureWebpack: {
    name: name,
    resolve: {
      alias: {
        '@': resolve('src')
      }
    },
    //此处就是
    externals:{
      'vue': 'Vue',
      'vue-router': 'VueRouter',
      'vuex': 'Vuex',
      'axios': 'axios',
      'element-ui':'ELEMENT',
      'js-cookie':'Cookies',
      'nprogress':'NProgress'
    }
  }
```

## 将import到的地方都要注释掉

store里的

```javascript
/*import Vue from 'vue'
import Vuex from 'vuex'*/
```

router里的 文件里面的Router需要替换成VueRouter

```javascript
/*import Vue from 'vue'
import Router from 'vue-router'*/
Vue.use(VueRouter)
const createRouter = () => new VueRouter({
  // mode: 'history', // require service support
  scrollBehavior: () => ({ y: 0 }),
  routes: constantRoutes
})
```

同理，引用的js都要把import去除掉