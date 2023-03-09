---
layout: post
title:  "fetch()与axios对比"
date:   2023-03-02 09:38:58 +0800
tags: vue
---

fetch()方法在2015年的时候可用，前端程序员终于有了除xhr（XMLHttpRequest）之外的官方选择，但axios依旧受欢迎，npm下载量稳定增长，可见官方的fetch仍有不足之处。
axios是基于promise的发送网络请求的第三方库，他的底层基于xhr实现，因此兼容性更好，基本使用如下
```
axios.get('https://example.com/')
  .then(response => console.log(response.data));
```
axiios返回数据默认是json格式,我们通过response的data属性获取返回值。

还可以传入第二个参数配置选项
```
axios(url, {
  // configuration options
})
```
例如
```
axios(url, {
  method: 'post',
  timeout: 1000,
  headers: {
    "Content-Type": "application/json",
  },
  data: {
    property: "value",
  },
})
```
去掉post时`axios(url)`默认执行get方法.



我们来看看js原生fetch()方法的基本使用,和axios很像，某些参数属性有差异：
```
fetch('https://example.com/')
  .then(response => response.json())
  .then(console.log);
```
使用fetch()的时候，当header到达时，就认为response已经达到了，但相应的body还没到，所以response.json()返回一个promise，这时候then()返回的promise我们称之已经resolved，但还没有fulfill，当body到达时，第一个promise立刻fulfill，then()返回的promise也立即fulfill。
传入两个参数的情形：
```
fetch(url, {
  method: 'POST',
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({  //'body' should be 'data' in axios
    property: "value",
  }),
})
```
在post数据时，fetch api需要使用JSON.stringify方法将数据转化为字符串，axios则自动转变。

在网络请求发生错误时，axios与fetch()的不同在于即使有http错误出现，fetch()不会reject一个promise，而是用then()处理错误
```
fetch('https://codilime.com/')
  .then(response => {
    if (!response.ok) {
      throw Error(`HTTP error: ${response.status}`);
    }
    return response.json();
  })
  .then(console.log)
  .catch((err) => {
    console.log(err.message)
  });
  ```
  axios则不同，它会拒绝状态码在200-299之外的response。
  ```
  axios.get('https://codilime.com/')
  .then(response => console.log(response.data))
  .catch((err) => {
    if (err.response) {
      // The request was made, but the server responded with a status code that falls out of the 2xx range
      const { status } = err.response;

      if (status === 401) {
        // The client must authenticate itself to get the requested response
      }
      else if (status === 502) {
        // The server got an invalid response
      }
      ...
    }
    else if (err.request) {
      // The request was made, but no response was received
    } 
    else {
      // Some other error
    }
  });
  ```
