## 前言

好久没写博客了，不是我偷懒啊，是我最近刚换工作，忙着熟悉业务和熟悉新生活，还没找到合适的步伐继续前进。

今天要分享的是`axios`源码。起因是我本来想学`ts`，然后某课上面有使用`ts`开发一个`axios`，再然后我看到中间我就看不懂了，干脆自己来研究一下`axios`的源码，所以就有了这篇总结了。

## 功能分析

参考文档和源码，我总结出来大概以下几个功能：

- 发送请求，支持浏览器环境和`Node`环境；
- 创建`Axios`实例；
- 有请求和响应拦截器；
- 支持转换请求数据和响应数据
- 支持取消功能；
- 支持并发功能等。

## 目录分析

```bash
- lib
	- adapters      # 请求封装，适配浏览器和node环境
	- cancel        # 跟取消相关
	- core          # 核心内容
	- helpers       # 工具类函数（可以独立于axios使用）
	- axios.js      # index.js中就是使用了这个文件
	- default.js    # 默认配置项
	- utils.js      # 工具类函数
```

## 源码分析

### 入口文件

从`package.json`中可以看出，入口文件在`index.js`中，而`index.js`的文件引入了`lib/axios.js`：

```js
function createInstance(){ /** ..  */  } 

// 创建Axios实例
var axios = createInstance(defaults);
// 便于后续继承Axios
axios.Axios = Axios;

// 跟取消请求有关
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// 跟并发请求有关
axios.all = function all(promise){ /** ..  */ }
axios.spread = require('./helpers/spread');

// 判断请求错误
axios.isAxiosError = require('./helpers/isAxiosError');

module.exports = axios;
module.exports.default = axios;
```

### 工具方法

在阅读源码的过程中，强烈推荐大家可以先去把`utils.js`和`helpers`文件夹中的代码都过一遍：

```js
// utils.js

// ...

module.exports = {
  isArray: isArray, 
  isArrayBuffer: isArrayBuffer,
  isBuffer: isBuffer,
  isFormData: isFormData,
  isArrayBufferView: isArrayBufferView,
  isString: isString,
  isNumber: isNumber,
  isObject: isObject,
  isPlainObject: isPlainObject,
  isUndefined: isUndefined,
  isDate: isDate,
  isFile: isFile,
  isBlob: isBlob,
  isFunction: isFunction,
  isStream: isStream,
  isURLSearchParams: isURLSearchParams,
  isStandardBrowserEnv: isStandardBrowserEnv,
  forEach: forEach,   // 遍历对象或者数组
  merge: merge,       // 将A、B两个变量进行合并
  extend: extend,     // A继承B的属性
  trim: trim,
  stripBOM: stripBOM
};
```

`utils.js`主要提供了一些`is`开头的函数来判断是否是某种类型或者格式的变量，还有遍历、合并、继承等工具函数。

`helpers`文件中有：

```bash
- bind.js          # 封装了一个类似bind的方法
- buildURL.js      # 将params参数拼接在url上
- combineURLs.js   # 组合获得新的url
- cookies.js       # 针对不同环境，对cookie的读、写、删除的封装
- deprecatedMethod.js # 对废弃方法的警告
- isAbsoluteURL.js # 判断是否是绝对路径
- isAxiosError.js  # 判断是否是Axios执行过程中抛出的异常
- isURLSameOrigin.js # 针对不同环境，判断是否跨域
- normalizeHeaderName.js # 格式化header的key
- parseHeaders.js  # 格式化header的参数并封装在一个对象中
- spread.js  # axios.spread的封装
- validator.js # 校验方法
```

### 默认Config

在创建实例的时候，我们会传递一个默认`config`进去：

```js
var defaults = {

  transitional: {
    silentJSONParsing: true, // 是否忽略JSON Parse的报错
    forcedJSONParsing: true, // resposeType不是json的情况下是否将响应值改为false
    clarifyTimeoutError: false // 超时的时候是否用ETIMEDOUT代替ECONNABORTED
  },

  adapter: getDefaultAdapter(),  // 请求方法（浏览器环境就是发起XMLHttpRequest)
  
  // 对请求数据进行转换
  transformRequest: [function transformRequest(data, headers) {
    normalizeHeaderName(headers, 'Accept');
    normalizeHeaderName(headers, 'Content-Type');

    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) {
      return data;
    }
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }
    if (utils.isURLSearchParams(data)) {
      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');
      return data.toString();
    }
    if (utils.isObject(data) || (headers && headers['Content-Type'] === 'application/json')) {
      setContentTypeIfUnset(headers, 'application/json');
      return stringifySafely(data);
    }
    return data;
  }],
  
  // 对响应式数据进行转换
  transformResponse: [function transformResponse(data) {
    var transitional = this.transitional;
    var silentJSONParsing = transitional && transitional.silentJSONParsing;
    var forcedJSONParsing = transitional && transitional.forcedJSONParsing;
    var strictJSONParsing = !silentJSONParsing && this.responseType === 'json';

    if (strictJSONParsing || (forcedJSONParsing && utils.isString(data) && data.length)) {
      try {
        return JSON.parse(data);
      } catch (e) {
        if (strictJSONParsing) {
          if (e.name === 'SyntaxError') {
            throw enhanceError(e, this, 'E_JSON_PARSE');
          }
          throw e;
        }
      }
    }

    return data;
  }],

  // 超时时间，如果为0代表不设置
  timeout: 0,

  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',

  maxContentLength: -1,
  maxBodyLength: -1,
  
  // `validateStatus` 定义对于给定的HTTP 响应状态码是 resolve 或 reject  promise 。
  // 如果 `validateStatus` 返回 `true` (或者设置为 `null` 或 `undefined`)，
  // promise 将被 resolve; 否则，promise 将被 rejecte
  validateStatus: function validateStatus(status) {
    return status >= 200 && status < 300;
  }
};

defaults.headers = {
  common: {
    'Accept': 'application/json, text/plain, */*'
  }
};

utils.forEach(['delete', 'get', 'head'], function forEachMethodNoData(method) {
  defaults.headers[method] = {};
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE);
});

module.exports = defaults;
```

### Axios类

在`createInstance()`中`new Axios()`，`Axios`的源码在`core/Axios.js`：

```js
// core/Axios.js
function Axios(instanceConfig) {
  // 配置项
  this.defaults = instanceConfig;
  // 拦截器
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}

// 核心！！！ 请求方法
Axios.prototype.request = function request(config) {}

// 获得uri
Axios.prototype.getUri = function getUri(config) {}

// 根据request扩展出不同方法下的请求
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});

module.exports = Axios;
```

### 拦截器

在看`request`实现之前，先重点看看拦截器：

```js
function Axios(instanceConfig) {
  // 配置项
  this.defaults = instanceConfig;
  // 拦截器
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```

`InterceptorManager`定义在`core/InterceptorManager.js`中：

```js
function InterceptorManager() {
  this.handlers = [];
}

// use是添加拦截器，其中fulfilled和rejected对应的就是promise的那两个
InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected,
    synchronous: options ? options.synchronous : false,
    runWhen: options ? options.runWhen : null
  });
  return this.handlers.length - 1;
};

// eject删除拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// forEach遍历拦截器
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};

module.exports = InterceptorManager;
```

拦截器的核心就是定义了一个`handlers`数组，每次新增一个拦截器，就新增一个`fulfilled、rejected、synchronous和runWhen`的对象，然后放置到`handlers`数组中，如果要删除，将该对应的`handlers[index]`置为空，这样遍历的时候，就不会被执行了。

### request请求

```js
Axios.prototype.request = function request(config) {
  // 支持axios(url[, config])
  if (typeof config === 'string') {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
  // 支持axios(config)
    config = config || {};
  }
  
  // 将用户的config和默认config进行合并
  config = mergeConfig(this.defaults, config);

  // 设置method
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = 'get';
  }
  
  // 配置transitional
  var transitional = config.transitional;

  if (transitional !== undefined) {
    validator.assertOptions(transitional, {
      silentJSONParsing: validators.transitional(validators.boolean, '1.0.0'),
      forcedJSONParsing: validators.transitional(validators.boolean, '1.0.0'),
      clarifyTimeoutError: validators.transitional(validators.boolean, '1.0.0')
    }, false);
  }

  // 添加请求拦截器，顺序是倒叙的
  // 比如 调用 A B C接口，那么请求拦截器的顺序为[C、B、A]
  var requestInterceptorChain = [];
  var synchronousRequestInterceptors = true;
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    if (typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false) {
      return;
    }

    synchronousRequestInterceptors = synchronousRequestInterceptors && interceptor.synchronous;

    requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  
  // 添加响应拦截器，顺序是倒正叙的
  // 比如 调用 A B C接口，那么请求拦截器的顺序为[A、B、C]
  var responseInterceptorChain = [];
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });

  var promise;
  
  // 如果是异步的话
  if (!synchronousRequestInterceptors) {
    var chain = [dispatchRequest, undefined];

    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain = chain.concat(responseInterceptorChain);

    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }

  // 如果是同步的话
  // 先执行 请求拦截器
  // 执行请求
  // 执行 响应拦截器
  var newConfig = config;
  while (requestInterceptorChain.length) {
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }

  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }

  while (responseInterceptorChain.length) {
    promise = promise.then(responseInterceptorChain.shift(), responseInterceptorChain.shift());
  }

  return promise;
};
```

### Axios直接发起请求

上面封装了`request`，这就不难理解`createInstance`了：

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  // 把request直接赋值给instance
  var instance = bind(Axios.prototype.request, context);

  utils.extend(instance, Axios.prototype, context);

  utils.extend(instance, context);

  instance.create = function create(instanceConfig) {
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };

  return instance;
}
```

### 取消请求

源码中跟取消相关的一共就三个文件：

```js
// cancel/Cancel.js

// Cancel就是一个含有message属性的类
function Cancel(message) {
  this.message = message;
}

Cancel.prototype.toString = function toString() {
  return 'Cancel' + (this.message ? ': ' + this.message : '');
};

Cancel.prototype.__CANCEL__ = true;

module.exports = Cancel;
```

```js
// cancel/CancelToken.js

function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}


CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
};

CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};

module.exports = CancelToken;
```

```js
// cancel/isCancel.js

module.exports = function isCancel(value) {
  return !!(value && value.__CANCEL__);
};

```

其实我没用过取消请求的功能，在看`React`相关的小册子的时候，那个作者提到了一个实际案例：

> 实时搜索的过程中，除了使用防抖/节流，还可以搭配取消请求，因为最后搜索的时候，前一个请求可能还没结束，这时候可以在发起请求之前，取消前面的请求。

这里给一个其他作者写的实际用例：[axios中取消请求及阻止重复请求的方法](https://blog.csdn.net/harsima/article/details/93717217)



---

如有错误欢迎指出，感谢阅读~
