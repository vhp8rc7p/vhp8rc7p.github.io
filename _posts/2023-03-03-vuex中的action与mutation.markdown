---
layout: post
title:  "vuex中的action与mutation"
date:   2023-03-03 09:38:58 +0800
tags: 
---

vuex是受到react的redux启发专门为vue.js设计的状态管理模式及库(state management pattern + library),它为所有组件充当了集中式存储的角色，同时存储中的数据改变是可预测的。

在vue的单项数据流中，当很多视图(view)依赖于同一个状态时，应用就会变得很复杂，当然我们可以通过重复写大量prop传递数据来解决，或者直接引用父子组件或全局事件总线。

vuex就是来解决这个问题的，它把共享的数据从组件中抽离出来单独管理，组件树形成了一个巨大的视图。
![](../assets/2023-03-03-vuex中的action与mutation/vuex.png)