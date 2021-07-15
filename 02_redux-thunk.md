## 前言

前面学习`redux`时，学到了`applyMiddleware`云里雾里，所以这次学习一下`redux-thunk`的源码，希望有助于深入理解`applyMiddleware`的源码实现。

## redux-thunk的作用

在`redux`中我们使用`dispatch(action)`时，`action`必须是一个简单对象，但是如果我们希望`dispacth`时可以进行各种逻辑处理，比如异步操作时，此时`action`会是一个函数，那么我们就需要借助`redux-thunk`这样的中间件了。

```js
export changeList = (data) =>({
    type: 'CHANGE_LIST',
    data
})

export const getList = (id) => {
    return async(dispatch, getState) => {
        const state = getState();
        const bookName = state.bookName;
        let res = await axios.get('xxxx?id' + id + '&bookName=' + bookName);
        dispatch(changeList(res.data));
    }
}

dispacth(getList)
```

## 中间件的思想

在不使用中间件时：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210716070942581.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70)
使用了中间件之后：

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-nwmRl9eA-1626390355261)(redux-thunk.assets/4116027-993b5e2ebc72a5a1.png)\]](https://img-blog.csdnimg.cn/20210716071009615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70)派发给`store`的`action`会经过中间件一层层处理，最终才会到达`store`。中间件的顺序跟`action`的处理顺序是密切相关的，只有前面的中间件完成任务，后面的中间件才有机会继续处理`action`。每个中间件都有自己的“熔断”处理,当它认为这个 action 不需要后面的中间件进行处理时，后面的中间件也就不能再对这个 action 进行处理了。

## 中间件基本架构

前面我们学习写`logger`中间件的时候提到，中间件函数的基本架构是这样的：

```js
const middlewarea = ({dispatch, getState}) => (next) => (action) => {
   next(action);
}
```

中间件函数接受两个参数，分别是`dispatch`和`getState`，该函数返回的函数接收一个`next`类型的参数，如果调用了内部这个函数，那么代表中间件完成了自己的职能，并将控制权交给下一个中间件。即`(action) => next(action)`会被交给下一个中间件。

`(action) => {}`这个函数可以进行多种操作：

- 调用`dispatch`派发一个新的`action`对象；
- 调用`getState`获得当前`store`上的其他状态；
- 调用`next`告诉`redux`当前中间件调用完毕，可以调用下一个中间件了；
- 访问`action`对象的所有数据。

## redux-thunk的源码

```js
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

内部判断`action`的类型，如果是非函数类型，那么直接`next(action)`调用下个中间件，如果是函数类型，那么先把`action`这个函数执行，其中`action`的参数分别为`dispatch、getState和arguments`。

到这里我们是不是还是觉得很抽象，接下来我们结合`applyMiddleware`来分析一下。

## 理解执行过程

上一节我们说到`createStore`内部是返回一个`store`对象，如果在内部遇到`enhancer`，即我们使用了中间件的话，那么就直接返回`enhancer`的执行结果，所以`enhancer`执行之后的函数必须为`store`对象。

```js
// ...
return enhancer(createStore)(reducer, preloadedState);
```

再加上

```js
const store = createStore(reducer, preloadedState, applyMiddleware(thunk))
```

那么现在可以确定的是`applyMiddleware`函数的基本架构应该是这样的：

```js
export const applyMiddleware = (...middlewreas){
    return (createStore) => (reducer, preloadedState) => {
        const store = createStore(reducer, preloadedState);
        // 处理dispatch,并将新的dispatch替换store中的dispacth
        return store;
	}
}
```

如何处理`dispatch`，我们继续看：

```js
export default function applyMiddleware(...middlewares){
    return (createStore) => (reducer, preloadedState) => {
      const store = createStore(reducer, preloadedState)
      // 如果在中间件构造过程中调用，抛出错误提示
      let dispatch = () => {
        throw new Error(
          'Dispatching while constructing your middleware is not allowed. ' +
            'Other middleware would not be applied to this dispatch.'
        )
      }
      const middlewareAPI = {
        getState: store.getState,
        dispatch: (action, ...args) => dispatch(action, ...args)
      }
      // 存放多个middleware调用的结果
      const chain = middlewares.map(middleware => middleware(middlewareAPI))
      // 用compose整合chain数组，并赋值给dispatch
      // compose(f1, f2, f3) = compose(f1(f2(f3())))
      dispatch = compose(...chain)(store.dispatch)
	  // 将新的dispatch替换原先的store.dispatch
      return {
        ...store,
        dispatch
      }
    }
}
```

我们尝试来捋一遍当执行`applyMiddleware(thunk))`的时候会发生什么：

```js
// const thunk = createThunkMiddleware();
// 即
thunk = ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
};
```

```js
// 在applyMiddlewrea中会执行
// const chain = middlewares.map(middleware => middleware(middlewareAPI))

// 即
const reduxFn = (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
}
const chain = [reduxFn];
```

```js
// 在applyMiddlewarea中继续执行
// dispatch = compose(...chain)(store.dispatch)

// 前面对compose的源码进行过分析，此时即为：
dispatch = ((next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
})(store.dispatch)

// 即使用了redux-thunk后，增强了dispacth的能力，既可以接受简单对象，也可以接受函数
dispacth = (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    
    // 这个next即dispatch
    return next(action)
}

// 所以如果action不是一个函数的话，就直接执行
return dispatch(action)

// 如果action是一个函数的话，那么就运行这个函数
return action(dispatch, getState, extraArgument);
```

```js
// 前面我们写了这个栗子
export const getList = (id) => {
    return async(dispatch, getState) => {
        // 获得store中其他state
        const state = getState();
        const bookName = state.bookName;
        let res = await axios.get('xxxx?id' + id + '&bookName=' + bookName);
        dispatch(changeList(res.data));
    }
}

// 此时如果执行dispacth(getList)的话，那么就是
(async(dispatch, getState) => {
    const state = getState();
    const bookName = state.bookName;
    let res = await axios.get('xxxx?id' + id + '&bookName=' + bookName);
    dispatch(changeList(res.data));
})(dispatch, getState)
```

## 总结

`redux-thunk`的源码真的好短！从源码理解来说不太难，但是结合`applyMiddleware`来跑一边，还是有点绕。这次通过看源码，发现了自己之前开发的时候，年前不懂事，不知道函数`action`内部可以通过`getState()`获得`store`的值.......现在学到了。

## 参考

- [深入理解Redux中间件](https://www.jianshu.com/p/ae7b5a2f78ae)
- [我的源码阅读之路：redux源码剖析](https://segmentfault.com/a/1190000016460366)

