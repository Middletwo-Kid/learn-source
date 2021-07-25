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

## 浅比较

| 值       | ===   | is(x,y) |
| -------- | ----- | ------- |
| NaN、NaN | false | true    |
| 0、+0    | true  | true    |
| 0、-0    | true  | false   |
| +0、-0   | true  | false   |

```js
// Object.is可以对基本数据类型:null,undefined,number,string,boolean做出非常精确的比较，
// 但是对于引用数据类型是没办法直接比较的。
function is(x, y) {
  if (x === y) {
    // 处理0、+0和-0的情况
    return x !== 0 || y !== 0 || 1 / x === 1 / y
  } else {
    // 处理NaN的情况
    return x !== x && y !== y
  }
}

export default function shallowEqual(objA, objB) {
  // 先用is来判断两个变量是否相等
  if (is(objA, objB)) return true
    
  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    return false
  }
  
  // 遍历两个对象上的键值对，一次进行比较
  const keysA = Object.keys(objA)
  const keysB = Object.keys(objB)

  if (keysA.length !== keysB.length) return false

  for (let i = 0; i < keysA.length; i++) {
    if (
      // Object.prototype.hasOwnProperty 和 in 运算符不同，该方法会忽略掉那些从原型链上继承到的属性。
      !Object.prototype.hasOwnProperty.call(objB, keysA[i]) ||
      !is(objA[keysA[i]], objB[keysA[i]])
    ) {
      return false
    }
  }

  return true
}
```
