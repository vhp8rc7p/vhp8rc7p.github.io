---
layout: post
title:  "深度监视(deep watcher)"
date:   2023-02-28 09:38:58 +0800
tags: JavaScript 学习笔记

---

有如下代码，我们在data属性里设置一个item对象数组，watch属性监视item，item被修改时console.log打印信息，再设置两个定时器，在1s和2s后回调分别直接通过$data直接修改item和通过调用Vue上的Set方法修改。
```
var vm = new Vue({
            el: '#app',
            watch: {
                item: {
                    handler(val, oldVal) {
                        console.log('Item Changed')
                        console.log(val)
                    }
                }
            },
            data: {
                item: [{ foo: 'foo' }]
            }
        })

        setTimeout(() => {
            vm.$data.item[0]['foo'] = 'bar';
        }, 1000)

        setTimeout(() => {
            Vue.set(vm.$data.item, 1, { 'bar': 'bar' });
        }, 2000)
```
首先关闭深度监视，控制台的输出如下
![](../assets/2023-02-27-深度监视(deep%20watcher)/console.png "console")
可以看到，vue只监视到了通过Set方法修改，通过$data属性修改没有监视到。
在watch属性下设置deep:true，此时可以检测到两次
![](../assets/2023-02-27-深度监视(deep%20watcher)/console1.png)