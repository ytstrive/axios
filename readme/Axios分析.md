#### 构造函数

```javascript
// lib/core/Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  //初始化request、response拦截器实例
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```

构造函数内包含了配置项、初始化 request、response 拦截器实例（常用来处理登录、异常消息(后面详细介绍拦截器).

#### request 方法

```javascript
Axios.prototype.request = function request(config) {
  /**
   * 如果参数config是字符串类型,则表示开发者使用的是
   * 1.axios(url)
   * 2.axios(url, [config])
   * 这两种方式其中一种方式调用,示例如下:
   * 1.axios('/user/12345')//默认get请求
   * 2.axios('/user',{params:{id:12345}})
   */
  if (typeof config === 'string') {
    //此处处理开发者使用axios(url)这种方式调用,只传递一个参数(url),第二个参数arguments[1]为undefined的情况将config默认为对象(简单理解将参数url合并到config)
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }
  //合并默认配置项与开发者自定义config(将合并之后的值重新赋值给开发者自定义config)
  config = mergeConfig(this.defaults, config);

  //判断method类型(此处优先级 开发者自定义config>默认配置项this.defaults.method>默认get ）
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = 'get';
  }

  var chain = [dispatchRequest, undefined];
  //将config作为参数传入Promise.resolve内,返回一个Promise对象
  var promise = Promise.resolve(config);
  //把request请求拦截器放入chain数组的起始位置
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  //把response响应拦截器放入chain数组结束位置
  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });
  //依次循环数组内的request拦截器、dispatchRequest(服务请求)、response响应拦截器
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }
  //最终返回promise对象
  return promise;
};
```

#### 其它

```javascript
//lib/core/Axios.js
utils.forEach(
  ['delete', 'get', 'head', 'options'],
  function forEachMethodNoData(method) {
    Axios.prototype[method] = function(url, config) {
      return this.request(
        utils.merge(config || {}, {
          method: method,
          url: url
        })
      );
    };
  }
);

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function(url, data, config) {
    return this.request(
      utils.merge(config || {}, {
        method: method,
        url: url,
        data: data
      })
    );
  };
});
```

从源码可以看出对外提供的多种调用方式,其实全部是挂载到原型上的方法,最终统一调用原型上的 request 方法,唯一区别是 post、put、patch 这些方法第二个参数需要传递 data 参数.

#### 问答

> 构造函数做了哪些

- `this.defaults` 接收参数,初始化默认配置项
- `this.interceptors` 初始化拦截对象,并且创建 request、response 拦截器的实例对象

构造函数做这两件事情目的是为 Axios.prototype.request 方法做准备.

> 原型方法 request 中声明 `var chain` 变量,数组初始化为什么要有 `undefined`

```javascript
//lib/core/Axios.js
var chain = [dispatchRequest, undefined];
while (chain.length) {
  promise = promise.then(chain.shift(), chain.shift());
}
```

**查看 `promise.then` 的语法,最多可以有 2 个参数`.then(onFulfilled[, onRejected])`**

那么就不难理解 `undefined` 作为.then 方法的第二个参数用来作为失败的回调.
**同理`chain.unshift(interceptor.fulfilled, interceptor.rejected)`request 请求拦截器和`chain.push(interceptor.fulfilled, interceptor.rejected);` response 响应拦截器向 chain 添加元素时均是新增两个元素(fulfilled、rejected)处理成功、失败的回调.**

```javascript
//示例代码
var chain = [
  function onFulfilled(value) {
    console.log(value);
  },
  function onRejected(value) {
    console.log(value);
  }
];
var promise = new Promise(function(resolve, reject) {
  resolve('Success');
  //or
  //reject('Error');
});
promise.then(chain[0], chain[1]);
```

> request、response 拦截器为什么分别使用`unshift` `push`方法添加元素

```javascript
//lib/core/Axios.js
this.interceptors.request.forEach(function unshiftRequestInterceptors(
  interceptor
) {
  chain.unshift(interceptor.fulfilled, interceptor.rejected);
});

this.interceptors.response.forEach(function pushResponseInterceptors(
  interceptor
) {
  chain.push(interceptor.fulfilled, interceptor.rejected);
});
```

首先我们需要知道一个流程:发送 request 请求->等待 response 响应,所以拦截器分别在此流程开始之前(request 拦截器)和结束之后(response 拦截器).

**request 拦截器->发送 request 请求->等待 response 响应->response 拦截器**

按照上面的流程,对应代码

```javascript
var chain = [dispatchRequest, undefined];
```

初始化数组已经包含了**发送 request 请求->等待 response 响应**此步骤(dispatchRequest),按照流程数组内还需要添加**request 拦截器**、**response 拦截器**

**添加 request 拦截器**

```javascript
//lib/core/Axios.js
this.interceptors.request.forEach(function unshiftRequestInterceptors(
  interceptor
) {
  chain.unshift(interceptor.fulfilled, interceptor.rejected);
});
```

根据流程**request 拦截器**是在**发送 request 请求**之前,所以使用`unshift`在数组起始位置添加元素

**`unshift()` 方法将一个或多个元素添加到数组的开头，并返回该数组的新长度(该方法修改原有数组)**

```javascript
//示例代码
var chain = [3, 4]; //3:请求响应
chain.unshift(1); //1:request拦截请求
//console.log(chain)1,3,4
```

**添加 response 拦截器**

```javascript
//lib/core/Axios.js
this.interceptors.response.forEach(function pushResponseInterceptors(
  interceptor
) {
  chain.push(interceptor.fulfilled, interceptor.rejected);
});
```

根据流程**response 拦截器**是在**等待 response 响应**之后也就是末尾,所以直接使用`push`在数组末尾位置添加元素

```javascript
//示例代码
var chain = [1, 3, 4];
chain.push(5); //5:response拦截请求
//console.log(chain)1,3,4,5
```

源码做的这一系列操作皆是让 **变量 chain=[...request 拦截器,请求响应,...response 拦截器]** 按照此流程添加元素.

如果在构造函数内只有一个实例对象`this.interceptor=new InterceptorManager()`,
无法区分请求与响应拦截器(无法实现以上流程步骤),所以`this.interceptors`对象即有 `request` 拦截器实例又有 `response` 拦截器实例的存在.

> 为什么拦截器是数组类型

无论 request、response 拦截器在实际应用场景中可能负责多个功能.

例如请求拦截器:

1. request.A token 校验
2. request.B loading 处理

所以`this.interceptors.request`、`this.interceptors.response`为数组类型.

> while 循环如何配合 promise.then 处理流程

使用过 promise 的都知道.then 方法的返回值是 Promise 对象,因此可以使用链式调用.
**如果对 Promise 感兴趣,请自行查找相关资料.**

```javascript
//promise.then链式调用示例代码
var promise1 = new Promise(function(resolve, reject) {
  resolve({ url: '/user/12345', method: 'get' });
});
promise1
  .then(function(value) {
    console.log(value); //{url: "/user/12345", method: "get"}
    return value;
  })
  .then(function(value) {
    //如果上一个then方法内没有显示的 return ;则输出undefined
    console.log(value); //{url: "/user/12345", method: "get"}
  });
```

我们知道了.then 方法可以链式调用,那么 while 循环则是对(流程)数组进行.then 链式调用的具体实现.

```javascript
//示例代码
var config = {
  url: '/user/12345',
  method: 'get',
  timeout: 3000
};
//request拦截成功回调
function requestOnFulfilled(config) {
  console.log('requestOnFulfilled', config);
  return config;
}
//request拦截失败回调
function requestOnRejected(reason) {
  console.log('requestOnRejected', reason);
  return Promise.reject(reason);
}
//response拦截成功回调
function responseOnFulfilled(response) {
  console.log('responseOnFulfilled', response);
  return response;
}
//response拦截失败回调
function responseOnRejected(reason) {
  console.log('responseOnRejected', reason);
  return Promise.reject(reason);
}
//请求
function dispatchRequest(config) {
  console.log('dispatchRequest', config);
  return config;
}
var chain = [dispatchRequest, undefined];
chain.unshift(requestOnFulfilled, requestOnRejected);
chain.push(responseOnFulfilled, responseOnRejected);
//chain=[requestOnFulfilled,requestOnRejected,request,undefined,responseOnFulfilled,responseOnRejected];
//将config作为参数传入Promise.resolve内,返回一个Promise对象
var promise = Promise.resolve(config);
while (chain.length) {
  promise = promise.then(chain.shift(), chain.shift());
}
// while循环分解
// promise
//   .then(requestOnFulfilled, requestOnRejected)
//   .then(dispatchRequest, undefined)
//   .then(responseOnFulfilled, responseOnRejected);
```

**`shift()` 方法从数组中删除第一个元素，并返回该元素的值。此方法更改数组的长度**

至此我们已经对 while、promise.then 处理流程有所了解,但需要思考以下几个问题:

- 为什么循环体内 promise 需要重新赋值`promise = promise.then(chain.shift(), chain.shift());`而不可以使用`promise.then(chain.shift(), chain.shift());`
- 把`var promise = Promise.resolve(config);`修改为`var promise = Promise.reject(config);`;requestOnRejected 函数内的`return Promise.reject(reason);`直接返回 `return reason;`结果会如何.

> config 配置项如何传递到拦截器内

```javascript
//lib/core/Axios.js
var promise = Promise.resolve(config);
```

首先我们看创建 Promise 对象的同时把 config 作为参数传递到.resolve 静态方法内,目的就是为了.then 方法的成功回调可以获取到该参数.
**简单理解使用.resolve 静态方法创建 Promise 对象时会把参数带入到.then 回调方法内,可查看 Promise 相关资料**

```javascript
//示例代码
var config = {
  url: '/user/12345',
  method: 'get',
  timeout: 3000
};
//request拦截成功回调
function requestOnFulfilled(config) {
  console.log('requestOnFulfilled', config);
  return config;
}
//request拦截失败回调
function requestOnRejected(reason) {
  console.log('requestOnRejected', reason);
  return Promise.reject(reason);
}
var promise = Promise.resolve(config);
var chain = [requestOnFulfilled, requestOnRejected];
//此处只模拟了请求拦截器,所以未使用while循环
promise.then(chain.shift(), chain.shift());
```

这样一来便可以在请求拦截器内获取到 config,同时也需要注意以下几点:

- 如果没有配置拦截器,config 是直接传递到了 dispatchRequest 内

```javascript
//示例代码
var config = {
  url: '/user/12345',
  method: 'get',
  timeout: 3000
};
//请求
function dispatchRequest(config) {
  console.log('dispatchRequest', config);
  return config;
}
var promise = Promise.resolve(config);
var chain = [dispatchRequest, undefined];
promise.then(chain.shift(), chain.shift());
```

- 请求拦截器成功回调必须显示的返回 config

如果配置了请求拦截器,需要注意的是在成功回调的函数内必须显示`return config`,原因在于.then 的链式调用如果没有显示返回,则向下调用的函数内无法获取参数.

```javascript
//示例代码
var config = {
  url: '/user/12345',
  method: 'get',
  timeout: 3000
};
//request拦截成功回调
function requestOnFulfilled(config) {
  console.log('requestOnFulfilled', config);
  return config;
}
//request拦截失败回调
function requestOnRejected(reason) {
  console.log('requestOnRejected', reason);
  return Promise.reject(reason);
}
//请求
function dispatchRequest(config) {
  console.log('dispatchRequest', config);
  return config;
}
var chain = [dispatchRequest, undefined];
chain.unshift(requestOnFulfilled, requestOnRejected);
var promise = Promise.resolve(config);
while (chain.length) {
  promise = promise.then(chain.shift(), chain.shift());
}
```

可以试一下 requestOnFulfilled 函数如果没有`return config`,dispatchRequest 函数的输出结果是什么.

以上示例代码 dispatchRequest 内返回的是 `return config`,由于只是简单的示例代码无法演示`return response`,其实在源码内是要返回 response 对象供向下的响应拦截器使用.

**再次强调请求拦截器成功回调必须显示的返回 config**
**同理响应拦截器成功回调必须显示的返回 response**

```javascript
//lisb/core/dispatchRequest.js
function dispatchRequest(config){
  ...
  return response;
}
```

以上是对 Axios 做了一个整体的了解,可以总结为以下几点:

- 构造函数
  - 默认配置项
  - 初始化拦截器
- request 方法
  - 合并默认配置项与开发者自定义配置项
  - 创建 Promise 对象
  - 整合拦截器、请求(梳理流程)
  - while 循环执行流程
  - 返回 Promise 对象
- 多种调用方式实现
  - 原型上挂载`delete` `get` `head` `options` `post` `put` `patch`方法
