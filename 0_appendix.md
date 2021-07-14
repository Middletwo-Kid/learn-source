> 记录一些从源码学到的，觉得有用的代码片段。

## isPlainObject
通过原型链来判断是否是简单对象，`redux`中用来判断`dispatch`中的`action`。
```js
export default function isPlainObject(obj: any): boolean {
  if (typeof obj !== 'object' || obj === null) return false

  let proto = obj
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto)
  }

  return Object.getPrototypeOf(obj) === proto
}
```

## Number.prototype.toString(Radix)
`Number.prototype.toString()`可以传入`2~36`的参数，默认是10。基于这个特性写一个获取指定长度的随机字符串的长度：
```js
function randomString(length){
  let str='';
  while(length>0){
    const fragment= Math.random().toString(36).substring(2);
    if(length>fragment.length){
      str+=fragment;
      length-=fragment.length;
    }else{
      str+=fragment.substring(0,length);
      length=0;
    }
  }
  return str;
}
```

## compose
组合多个函数，可以将函数的返回值当作下一个函数的参数，特点是从右到左执行。
```js
export default function compose(...funcs){
    if(funcs.length === 0) return arg => arg

    if(funcs.length === 1) return funcs[0]

    return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
