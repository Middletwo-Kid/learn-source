> 新技术迭代太快，最近学累了，想慢下来，所以开始找一些感兴趣的源码来研究，学习一下大神们的思想。技术会一直迭代，但是基础和思想的东西是相通的。
## Redux的基本使用

在自己实现`redux`之前，我们先来回归一下`redux`的基本用法。

```js
import { createStore } from 'redux';

// 定义初始值
const defaultState = {
    cash: 200
}

// 定义不同的action
const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

// 根据不同的action处理state
const reducer = (state = defaultState, action) => {
    const { type, data } = action;
    switch(type){
        case INCREMENT: return { ...state, cash: state.cash + data };
        case DECREMENT: return { ...state, cash: state.cash - data };
        default: return state;
    }
}

// 创建唯一的store
const store = createStore(reducer);

// 监听store的变化
const unsubscribe = store.subscribe(() => console.log(`余额：${store.getState()}`));

// 增加300块现金
store.dispatch({
    type: INCREMENT,
    data: 300
});

// 减少100块
store.dispatch({
    type: DECREMENT,
    data: 100
});

// 取消监听
unsubscribe();
```

`redux`的常规用法总结起来就是这几点：

- 定义关于`action`的常量，如`const INCREMENT = 'INCREMENT'`；
- 定义默认值，如`defaultState`；
- 定义`reducer`，根据不同的`action`中的`type`来修改`state`，**切记state不可以直接修改，即不可直接使用`state = action.data`**;
- 创建`store`， 通过`createStore`，传入默认值和`reuder`，获得一个全局的管理器；
- 使用`subscribe`监听`store`,并返回一个`unsubscribe`（可以类比使用`setTimeout`时返回的值）；
- 用户不可以直接修改`store`，所以如果要更改`store`中的`state`，需要使用`dispatch`，`dispatch`需要传入包含`type`和`data`的对象；
- 使用`unsubscribe`取消监听。

## Redux的数据流向

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233448324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70#pic_center)


- 当需要修改`store`中的`state`时，用户或者说是组件需要借助`Action Creators`；
- 通过`Action Creators`通过`dispatch`一个含有`type`的对象给`store`；
- `store`接收到只会，会将现在的`state`和接收到的对象交给`reducer`处理；
- `reducer`通过不同的`action.type`，将老的`state`加工处理成新的`state`并返回给`state`；
- `store`将新的`state`交会给组件。

## 基本工作

在实现`Redux`之前，我们先准备一下项目。先用脚手架创建一个`react`项目，（`redux`其实适用于你觉得需要操作状态的所有项目，不仅是`react`，还可以是`vue`等）。

```bash
npx create-react-app redux-demo
```

`src`文件下只保留`index.js`文件，然后修改：

```js
// index.js
let state = {
  cash: 100,
  people: 3
}

function render(state){
  const dom = document.getElementById('root');
  dom.innerHTML = `当前家庭人口有${state.people}人,
                    <br/>
                    当前现金有${state.cash}`;
}

render(state);

```

启动并运行：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233505514.png#pic_center)


我们在页面中渲染了`state`中的值，接下来我们要围绕这个`demo`来实现数据的监听、修改等操作，实现一个简单的`redux`。

## `createStore`的实现和源码分析

`redux`源码中，暴露出了几个方法，分别是：`createStore、combineReducers、bindActionCreators、applyMiddleware、compose`等。我们首先来实现`createStore`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233522153.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70#pic_center)


### 实现`createStore`

从上面的基本使用中我们可以知道`createStore`，支持传入两个参数，一个是`state`，一个是`reducer`：

```js
creatStore(state, reducer);
```

并且返回了一个新的对象，该对象包含几个方法：

```bash
- getState();
- dispatch();
- subscribe();
```

#### 实现getState和dispatch

顺着思路写出下面代码：

```js
// redux的实现
const createStore = (reuducer) => {
  let state;

  const getState = () => state;

  const dispatch = (action) => {
    state = reuducer(state, action);
  }

  return{
    getState,
    dispatch
  }
}
```

```js
// 使用
let defaultState = {
  cash: 100,
  people: 3
}

const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DECREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);

store.dispatch({
  type: INCREMENT,
  data: 300,
});

function render(state){
  const dom = document.getElementById('root');
  dom.innerHTML = `当前家庭人口有${state.people}人,
                    <br/>
                    当前现金有${state.cash}`;
}

render(store.getState());
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/202107142335493.png#pic_center)


#### 初始化时应该`dispatch`一次

修改一下代码：

```js
let defaultState = {
  cash: 100,
  people: 3
}

const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DECREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);

console.log(store.getState()); // undefined

```

当我们第一次使用`store.getState()`时，结果应该为`defaultState`而不是`undefined`，所以我们在`creatStore`内部应该先执行一次`dispatch`，好让现在的`state`等于`defaultState`的值。

```js
const createStore = (reuducer) => {
  let state;

  const getState = () => state;

  const dispatch = (action) => {
    state = reuducer(state, action);
  }
  
  // 新增
  dispatch({ type: '@@REDUX_INIT' });

  return{
    getState,
    dispatch
  }
}
```

现在就没问题了，当前`store.getState()`的值为`{cash: 100, people: 3}`。

#### 实现监听和取消监听

我们调整一下代码，先执行渲染，再`diapacth`改变值，看看结果：

```js
// ...

// 先渲染
render(store.getState());

// 后修改数据
store.dispatch({
  type: INCREMENT,
  data: 300,
});

console.log(store.getState());  // {cash: 400, people: 3}
```

虽然打印出来的`store.getState`的值发生了改变，但是此时页面上的结果为：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233604113.png#pic_center)


我们需要对`state`进行监听，所以`createStore`返回的对象中会含有一个`subscribe`方法，实现上运用到了**发布订阅**这种设计模式。

```js
const createStore = (reducer) => {
    let state;
    // 新增
    let listeners = [];
    
    const getState = () => state;
    
    const dispatch = (action) => {
        state = reducer(state, action);
        // 新增
        listeners.forEach((fn) => fn());
    }
    
    dispatch({type: '@@REDUX_INIT'});
    
    // 新增
    const subscribe = (fn) => {
        listeners.push(fn);
        return () => {
            listeners = listeners.filter((item) => item !== fn);
        }
    }
    
    return {
        getState,
        dispatch,
        subscribe
    }
}
```

使用如下：

```js
let defaultState = {
  cash: 100,
  people: 3
}

const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DECREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);

function render(state){
  const dom = document.getElementById('root');
  dom.innerHTML = `当前家庭人口有${state.people}人,
                    <br/>
                    当前现金有${state.cash}`;
}

const unsubscribe = store.subscribe(() => {
  render(store.getState());
});


store.dispatch({
  type: INCREMENT,
  data: 300,
});                    // 此时页面上的数据为 cash: 400, people: 3

unsubscribe();

store.dispatch({
  type: INCREMENT,
  data: 300,
});

console.log(store.getState()); // 此时数据为 cash: 400, people: 3,但是页面上的数据并没有更新
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233616638.png#pic_center)


这样`createStore`就基本完成了，接下来我们来分析一下源码，并进行补充。

### 分析`createStore`

#### 暴露的方法

在源码中`createStore`暴露出来的方法一共有：

```bash
- dispatch
- subscribe
- getState
- replaceReducer
- $$observable
```

#### 接受的参数

源码中`createStore`可以接受三个参数：

```bash
- reducer             # 根据不同的action处理返回不同的state
- preloadState        # 初始化state,但是我们一般不适用
- enhancer            # 增强器
```

我们使用`redux-thunk`时候，我们就会用到`createStore`的第三个参数，例如：

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import reducer from './reducers';

const store = () => createStore(reducer, applyMiddleware(thunk));

export default store;
```

#### 参数的限制与兼容

在源码中，作者对参数的传入做了一些兼容和限制：

```js
export default function createStore(reducer, preloadedState, enhancer){
    // 使用enhancer作为增强器时，我们只能传入一个，如果你需要使用多个，得封装成一个
    if (
    (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
    (typeof enhancer === 'function' && typeof arguments[3] === 'function')
  ) {
    throw new Error(
      'It looks like you are passing several store enhancers to ' +
        'createStore(). This is not supported. Instead, compose them ' +
        'together to a single function. See https://redux.js.org/tutorials/fundamentals/part-4-store#creating-a-store-with-enhancers for an example.'
    )
  }
  
  // 类比于我们使用redux-thunk,我们并没有传递preloadedState， 那么此时就是
  // enhancer = preloadedState, preloadedState = undefined
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
    preloadedState = undefined
  }
    
  // 当传入了增强器时
  if (typeof enhancer !== 'undefined') {
    // 先要保证增强器是一个函数
    if (typeof enhancer !== 'function') {
      throw new Error(
        `Expected the enhancer to be a function. Instead, received: '${kindOf(
          enhancer
        )}'`
      )
    }
	
    // 调用增强器，后续我们分析redux-thunk源码就能知道enhancer为什么是这么使用了 todo
    return enhancer(createStore)(
      reducer,
      preloadedState
    )
  }
  
  //  确保reducer也是一个函数
  if (typeof reducer !== 'function') {
    throw new Error(
      `Expected the root reducer to be a function. Instead, received: '${kindOf(
        reducer
      )}'`
    )
  }
    
  // ...
}
```

#### 定义变量

接下来作者定义了几个变量：

```js
let currentReducer = reducer          // 存储reducer
let currentState = preloadedState     // 存储state
let currentListeners: []              // 当前订阅者列表
let nextListeners = currentListeners  // 新的订阅者列表
let isDispatching = false             // 是否在dispatch
```

#### `getState`

```js
function getState() {
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }

    return currentState
}
```

调用`getState`出现报错的情况一般是在`reducer`中处理`state`的时候：

```js
const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: 
          // 就是这行作死的代码
          console.log(store.getState()); 
          return { ...state, cash: state.cash + data };
    default: return state;
  }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233632586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70#pic_center)


所以我们在`dispatch`改变`state`时，不可以在`recuder`接受`dispatch`过来的`action`、返回新的的`state`前去读取`store.getState()`，这样会报错！！

#### `subscribe`

基本结构如下：

```js
function subscribe(listener) {
    // ...
    return () => {}
}
```

首先传参必须是一个函数：

```js
if (typeof listener !== 'function') {
  throw new Error(
    `Expected the listener to be a function. Instead, received: '${kindOf(
      listener
    )}'`
  )
}
```

其次在`dispatch`的过程中，我们不可以使用`subscribe`去监听`state`的变化：

```js
if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See https://redux.js.org/api/store#subscribelistener for more details.'
      )
 }
```

开始去订阅：

```js
let isSubscribed = true
```

浅拷贝`listeners`:

```js
ensureCanMutateNextListeners()
nextListeners.push(listener)
```

```js
function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
}
```

当`nextListeners`和`currentListeners`是同一个引用的时候，那么此时使用`slice`浅拷贝一个副本出来，然后将当前的新的订阅函数保存在`nextListners`中。

最后是取消订阅的逻辑：

```js
return function unsubscribe() {
  // isSubscribed代表处于监听状态，如果是false，就没必要取消了
  if (!isSubscribed) {
    return
  }
  
  // dispatch时也不可以取消监听
  if (isDispatching) {
    throw new Error(
      'You may not unsubscribe from a store listener while the reducer is executing. ' +
        'See https://redux.js.org/api/store#subscribelistener for more details.'
    )
  }
  
  // 将当前的监听状态取消
  isSubscribed = false
  
  // 将事件从订阅者中移除
  ensureCanMutateNextListeners()
  const index = nextListeners.indexOf(listener)
  nextListeners.splice(index, 1)
  currentListeners = null
}
```

#### `dispatch`

`dispatch`方法支持一个`action`的参数，这个参数必须是一个**简单的对象类型**：

```js
function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        `Actions must be plain objects. Instead, the actual type was: '${kindOf(
          action
        )}'. You may need to add middleware to your store setup to handle dispatching other values, such as 'redux-thunk' to handle dispatching functions. See https://redux.js.org/tutorials/fundamentals/part-4-store#middleware and https://redux.js.org/tutorials/fundamentals/part-6-async-logic#using-the-redux-thunk-middleware for examples.`
      )
    }
    
    // ...
    
    return action;
}
```

`isPlainObject`函数定义在`src/utils/isPlainObject.ts`下：

```js
export default function isPlainObject(obj) {
  if (typeof obj !== 'object' || obj === null) return false

  let proto = obj
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto)
  }
  
  // 对 getPrototypeOf 不熟悉可以理解成 __proto__
  // while(proto.__proto__ !== null){
  // 	proto = proto.__proto__  
  // }
  return Object.getPrototypeOf(obj) === proto
}

```

复习一下原型链：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233645531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MDg2OTgw,size_16,color_FFFFFF,t_70#pic_center)


`isPlainObject`先是判断是否是`object`对象，如果是的话，会一直去寻找他的实例原型：

```js
function Person(){}
let person = new Person();
isPlainObject(person);           // false

let obj = {name: '王花花'}
isPlainObject(obj)               //  true

isPlainObject({name: '王花花'})   // true
```

`person`的原型实例是`Person.prototype`，之后沿着原型链，才是`Object.prototype`。而`obj`或者是字面量对象`{name: '王花花'}`他们的原型实例都是`Object.prototype`。这些便是简单对象， **所谓的简单对象就是该对象的`__proto__`等于`Object.prototype`**。

了解完`isPlainObject`之后我们继续往下看：

```js
// diapatch(action)时，传参的对象action一定要有一个type属性
if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. You may have misspelled an action type string constant.'
      )
}

// 正在dispacth时，不可以继续dispatch
// 即两个用户dispacth修改同一个值时，这样是不允许的，一次只能操作一个
if (isDispatching) {
  throw new Error('Reducers may not dispatch actions.')
}
```

都没问题之后，就将`dispacth`中的`action`传递给`reducer`处理啦：

```js
try {
  isDispatching = true
  currentState = currentReducer(currentState, action)
} finally {
  isDispatching = false
}
```

在`dispatch`时，我们修改了值，所以我们要通知监听了这些值的订阅器：

```js
const listeners = (currentListeners = nextListeners)
for (let i = 0; i < listeners.length; i++) {
  const listener = listeners[i]
  listener()
}
```

在这里，有`const listeners = (currentListeners = nextListeners)`，还记得之前的`ensureCanMutateNextListeners`吗？

```js
function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
}
```

在`dispatch`时，`currentListeners`和`nextListeners`就是同一个引用了，而在`dispatch`后我们使用`subscribe`或者`unsubscribe`时，就会调用`ensureCanMutateNextListeners`将`currentListeners`浅拷贝出一个副本给`nextListeners`。

为什么要使用`currentListeners`和`nextListeners`，而不使用一个呢？

假设现在只有一个`currentListeners`，在我们`dispatch`时会触发`subscribe`去遍历`currentListeners`：

- 用户此时突然`subscribe`另一个事件的话，就会修改`currentListeners`，导致此时正在遍历`currentListeners`触发订阅事件出现问题，即订阅事件增加了；
- 另一种情况是，用户此时突然`unsubscribe`的话，`currentListeners`的事件数量将会减少，此时订阅顺序可能将被打乱。

所以为了事件的一致性，我们在`subscribe`或者`unsubscribe`时，要同时使用`currentListeners`和`nextListeners`。

**别忘了定义完`dispatch`**后，我们需要手动触发一次，让`currentState`等于`reducer`中默认的`state`值。

```js
dispatch({ type: ActionTypes.INIT })
```

#### `replaceReducer`

这个方法我基本没用到过，看名字就能知道就是替换`reducer`：

```js
function replaceReducer(nextReducer){
    if (typeof nextReducer !== 'function') {
      throw new Error(
        `Expected the nextReducer to be a function. Instead, received: '${kindOf(
          nextReducer
        )}`
      )
    }
    currentReducer = nextReducer;
    dispatch({ type: ActionTypes.REPLACE } as A)
    return store;
}
```

同理，在替换`reducer`时要确保新传入的`reducer`也是一个函数。

#### `$$observable`

虽然源码内部暴露了这个方法，但是我们调用不了，所以这里就不纠结它是干嘛的了。

#### 小结

我们虽然可以直接修改`store.getState()`返回的`state`，但是并不能引起视图更新，最根本的原因是需要使用`dispatch`，才会触发`store`内部的`state`更新从而在`store.getState()`中获取到最新的`state`而引起视图的更新。

## `combineReducers`的实现和源码分析

### 使用combineReducers

在实现之前我们先来看看`redux`中`combineReducers`的使用：

```js
import { createStore, combineReducers } from 'redux';

let defaultCashState = {
  cash: 100
}

let defaultPeopleState = {
  people: 3
}

const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

const INCREMENT_PEOPLE = 'INCREMENT_PEOPLE';          
const DECREMENT_PEOPLE = 'DECREMENT_PEOPLE';

const cashReducer = (state = defaultCashState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DECREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

const peopleReducer = (state = defaultPeopleState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT_PEOPLE: return { ...state, people: state.people + data };
    case DECREMENT_PEOPLE: return { ...state, people: state.people - data };
    default: return state;
  }
}

const reducer = combineReducers({cashReducer, peopleReducer});

let store = createStore(reducer);

console.log(store.getState());
```

这时候我们打印出来的结果为：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210714233702712.png#pic_center)


使用`combineReducers`合并`reducer`之后，他们各自的`state`最后在输出的时候，都被封装了，而且对应的`key`是各自对应的`reducer`的值。[combineReducers用法中有这么一段话：](https://www.redux.org.cn/docs/recipes/reducers/UsingCombineReducers.html)

> `combineReducers` 接收拆分之后的 reducer 函数组成的对象，并且创建出具有相同键对应状态对象的函数。这意味着如果没有给 `createStore` 提供预加载 state，输出 state 对象的 key 将由输入的拆分之后 reducer 组成对象的 key 决定。

那么如果我们在`createStore`传递`preloadedState `(第二个参数的话)：

```js
// 这里的key值应该与对应的reducer一致，不然会报错
let store = createStore(reducer, { cashReducer: {cash: 100}, peopleReducer: {people: 100}});
```

打印出来的结果为：

```js
cashReducer: {cash: 100}
peopleReducer: {people: 100}
```

### 实现`combineReducers`

从上面的栗子可以看出，使用`combineReducers`合并`reducer`之后,会返回一个新的`reducer`函数，并且`state`会被拆分，最后函数内部返回一个新的`state`（`reducer`函数内部也是根据`action`返回新的`state`）

```js
const combineReducers = (reducers) => {
    return (state ={}, action) => {
        let nextState = {};
        // todo
        return nextState;
    }
}
```

大致思路如下：

- 先获得各个`reducers`的`key`；
- 遍历`keys`，执行`reducer`，获得新的`state`；
- 将新的得到的`state`存储起来；
- 返回新的`state`。

```js
const combineReducers = (reducers) => {
  const keys = Object.keys(reducers);
  return (state = {}, action = {type: ''}) => {
    let nextState = {};
    for(let i = 0; i < keys.length; i++){
      const key = keys[i];
      let reducer = reducers[key];
      const currentState = state[key];
      nextState[key] = reducer(currentState, action);
    }
    return nextState;
  }
}
```

使用：

```js
let defaultPeopleState = {
  people: 3
}

const INCREMENT = 'INCREMENT';          
const DECREMENT = 'DECREMENT';

const INCREMENT_PEOPLE = 'INCREMENT_PEOPLE';          
const DECREMENT_PEOPLE = 'DECREMENT_PEOPLE';

const cashReducer = (state = defaultCashState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DECREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

const peopleReducer = (state = defaultPeopleState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT_PEOPLE: return { ...state, people: state.people + data };
    case DECREMENT_PEOPLE: return { ...state, people: state.people - data };
    default: return state;
  }
}

const reducer = combineReducers({cashReducer, peopleReducer});

let store = createStore(reducer);

console.log(store.getState()); // cashReducer: {cash: 200} peopleReducer: {people: 3}
```

### 分析`combineReducers`

#### 基本结构

跟猜想的一样，基本结构如下：

```js
export default function combineReducers(reducers) {
    return function combination(state, action){
        let nextState = {};
        // ...
        return nextState;
    }
}
```

#### 存储keys

源码中第一步也是先存储`reducers`中的各个`key`：

```js
export default function combineReducers(reducers) {
    const reducerKeys = Object.keys(reducers)
    const finalReducers: ReducersMapObject = {}
    for (let i = 0; i < reducerKeys.length; i++) {
        const key = reducerKeys[i]
        
		// 校验各个Key对应的reducer是否存在
        if (process.env.NODE_ENV !== 'production') {
          if (typeof reducers[key] === 'undefined') {
            warning(`No reducer provided for key "${key}"`)
          }
        }
		
        // 校验各个Key对应的reducer是否是函数
        if (typeof reducers[key] === 'function') {
          finalReducers[key] = reducers[key]
        }
    }
    // 最终合格的reducer[key]才会被保留下来
    const finalReducerKeys = Object.keys(finalReducers)

    // 将每次出现的Key存储在unexpectedKeyCache，确保每个key是唯一的
    let unexpectedKeyCache
    if (process.env.NODE_ENV !== 'production') {
    	unexpectedKeyCache = {}
    }

    let shapeAssertionError
    try {
        // 先触发每个reducer获得state的默认值，相当于前面部分，我们创建store时，先手动dispatch一下
        // 内部还对各个reducer进行了校验
    	assertReducerShape(finalReducers)
    } catch (e) {
    	shapeAssertionError = e
    }
    
    return function combination(state, action){
        let nextState = {};
        // ...
        return nextState;
    }
}
```
#### 校验`reducer`
```js
// 获得各个reducer的默认state,并对reducer进行校验
function assertReducerShape(reducers: ReducersMapObject) {
    Object.keys(reducers).forEach(key => {
        const reducer = reducers[key]
        // 先获得initialState，前面自己写的时候给action一个默认值，也是为了获得initialState
        const initialState = reducer(undefined, { type: ActionTypes.INIT })

        // 如果initialState为undefined,则说明你的reducer内部没有返回一个默认值
        if (typeof initialState === 'undefined') {
          throw new Error(
            `The slice reducer for key "${key}" returned undefined during initialization. ` +
              `If the state passed to the reducer is undefined, you must ` +
              `explicitly return the initial state. The initial state may ` +
              `not be undefined. If you don't want to set a value for this reducer, ` +
              `you can use null instead of undefined.`
          )
        }

        // 不能使用redux的私有的action.type类型
        if (
          typeof reducer(undefined, {
            type: ActionTypes.PROBE_UNKNOWN_ACTION()
          }) === 'undefined'
        ) {
          throw new Error(
            `The slice reducer for key "${key}" returned undefined when probed with a random type. ` +
              `Don't try to handle '${ActionTypes.INIT}' or other actions in "redux/*" ` +
              `namespace. They are considered private. Instead, you must return the ` +
              `current state for any unknown actions, unless it is undefined, ` +
              `in which case you must return the initial state, regardless of the ` +
              `action type. The initial state may not be undefined, but can be null.`
          )
        }
      })
}
```

这里说明一下`redux`内部的`action.type`类型，在`utils/actionTypes`文件下：

```js
const randomString = () =>
  Math.random().toString(36).substring(7).split('').join('.')

const ActionTypes = {
  INIT: `@@redux/INIT${/* #__PURE__ */ randomString()}`,
  REPLACE: `@@redux/REPLACE${/* #__PURE__ */ randomString()}`,
  PROBE_UNKNOWN_ACTION: () => `@@redux/PROBE_UNKNOWN_ACTION${randomString()}`
}

export default ActionTypes
```

内部导出了`INIT、REPLACE和PROBE_UNKNOWN_ACTION`三种，这些是不可以被用户使用的，所以用户在定义`action`时，`type`不可以是这三者之中的一个。

```js
- INIT：在创建store的时候，初始化state
- REPLACE：在替换reducer的时候，填充新state树
- PROBE_UNKNOWN_ACTION：在combineReducer.js中使用，随机类型探测
```

#### `getUnexpectedStateShapeWarningMessage`

```js
export default function combineReducers(reducers) {
    // ...
    
    return function combination(state, action){
        // 如果上面的reducer校验没通过，这里就抛出异常
        if (shapeAssertionError) {
      		throw shapeAssertionError
    	}
        if (process.env.NODE_ENV !== 'production') {
          // 对state进行校验：跟reducers的key对应上，必须是个简单对象
          const warningMessage = getUnexpectedStateShapeWarningMessage(
            state,
            finalReducers,
            action,
            unexpectedKeyCache
          )
          if (warningMessage) {
            warning(warningMessage)
          }
        }
    }
}
```

```js
function getUnexpectedStateShapeWarningMessage(
  inputState,
  reducers,
  action,
  unexpectedKeyCache
) {
  const reducerKeys = Object.keys(reducers)
  // state默认值，可以是由createStore第二个参数传递，也可以是reducer第一个参数传递
  const argumentName =
    action && action.type === ActionTypes.INIT
      ? 'preloadedState argument passed to createStore'
      : 'previous state received by the reducer'
  
  // reducers不符合规范
  if (reducerKeys.length === 0) {
    return (
      'Store does not have a valid reducer. Make sure the argument passed ' +
      'to combineReducers is an object whose values are reducers.'
    )
  }
  
  // 判断state是不是简单对象
  if (!isPlainObject(inputState)) {
    return (
      `The ${argumentName} has unexpected type of "${kindOf(
        inputState
      )}". Expected argument to be an object with the following ` +
      `keys: "${reducerKeys.join('", "')}"`
    )
  }

  // 确保state的key和reducer的key对应得上
  const unexpectedKeys = Object.keys(inputState).filter(
    key => !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key]
  )

  unexpectedKeys.forEach(key => {
    unexpectedKeyCache[key] = true
  })
  
  // 如果是替换reducer的action,跳过下面步骤，不打印异常信息
  if (action && action.type === ActionTypes.REPLACE) return

  if (unexpectedKeys.length > 0) {
    return (
      `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
      `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
      `Expected to find one of the known reducer keys instead: ` +
      `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
    )
  }
}
```

#### 获取新的`state`

```js
let hasChanged = false
const nextState = {}
for (let i = 0; i < finalReducerKeys.length; i++) {
  const key = finalReducerKeys[i]
  const reducer = finalReducers[key]
  const previousStateForKey = state[key]
  // 循环调用每个reducer,并传入对应的state,获得新的state
  const nextStateForKey = reducer(previousStateForKey, action)
  
  // nextStateForKey不能是undefined
  if (typeof nextStateForKey === 'undefined') {
    const actionType = action && action.type
    throw new Error(
      `When called with an action of type ${
        actionType ? `"${String(actionType)}"` : '(unknown type)'
      }, the slice reducer for key "${key}" returned undefined. ` +
        `To ignore an action, you must explicitly return the previous state. ` +
        `If you want this reducer to hold no value, you can return null instead of undefined.`
    )
  }
  // 将每个reducer返回的state存放在对应的nextState中
  nextState[key] = nextStateForKey
  // 这里巧妙地运用了hasChanged，判断每次dispatch时，前后的值是否一致，一致的话就不更新啦~
  hasChanged = hasChanged || nextStateForKey !== previousStateForKey
}
hasChanged =
  hasChanged || finalReducerKeys.length !== Object.keys(state).length
return hasChanged ? nextState : state
```

#### 小结

由`combineReducers`合成的`reducers`，在`dispatch action` 的时候是否调用了内部所有的 `reducer`，从源码可以看出，这是肯定的。

## `bindActionCreators`的实现和源码分析

### 使用`bindActionCreators`

```js
import { createStore, bindActionCreators } from 'redux';

let defaultState = {
  cash: 200
}

const INCREMENT = 'INCREMENT';  
const DESCREMENT = 'DESCREMENT'; 

const addAction = (data) => ({
  type: INCREMENT,
  data
})

const descrementAction = (data) => ({
  type: DESCREMENT,
  data
})

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DESCREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);

// store.dispatch(addAction(200));
// store.dispatch(descrementAction(1));

const add = bindActionCreators(addAction, store.dispatch);
const minus = bindActionCreators(descrementAction, store.dispatch);
add(200);
minus(1);

console.log(store.getState());

```

之前我们都是用`dispatch(createAction())`来修改`state`，如果你觉得每次都要写`dispatch`感觉麻烦的时候，可以使用`bindActionCreators`，上面的写法还可以修改为：

```js
// actionType.js
export const INCREMENT = 'INCREMENT';  
export const DESCREMENT = 'DESCREMENT'; 
```

```js
// actionCreators.js
import { INCREMENT, DESCREMENT } from './actionType';

export const add = (data) => ({
  type: INCREMENT,
  data
})

export const decrement = (data) => ({
  type: DESCREMENT,
  data
})
```

```js
// index.js
import { createStore, bindActionCreators } from 'redux';
import { INCREMENT, DESCREMENT } from './actionType';
import * as actionCreators from './actionCreators';

let defaultState = {
  cash: 200
}

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DESCREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);
const handlerCash = bindActionCreators(actionCreators, store.dispatch);

handlerCash.add(200);
handlerCash.decrement(1);

console.log(store.getState());
```

### 实现`bindActionCreators`

由上面的使用我们可以分析出，`bindActionCreators`内部是返回了一个函数，这个函数内部还调用了`dispatch(actionCreators(xxx))`：

```js
const bindActionCreators = (actionCreators, dispatch) => {
  return (...args) => {
    dispatch(actionCreators(...args))
  }
}
```

测试一下：

```js
// ...上面是createStore和bindActionCreators的实现

let defaultState = {
  cash: 200
}

const INCREMENT = 'INCREMENT';  
const DESCREMENT = 'DESCREMENT'; 

const addAction = (data) => ({
  type: INCREMENT,
  data
})

const descrementAction = (data) => ({
  type: DESCREMENT,
  data
})

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DESCREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);
const add = bindActionCreators(addAction, store.dispatch);
const minus = bindActionCreators(descrementAction, store.dispatch);
add(200);
minus(1);

console.log(store.getState());

```

竟然没问题，那么再测试上面的另一种写法：

```js
// ...
const handlerCash = bindActionCreators(actionCreators, store.dispatch);
// ...
```

是的报错了，毕竟实现的过程中我们只返回了一个函数。

```js
const bindActionCreators = (actionCreators, dispatch) => {
  if(typeof actionCreators === 'function'){
    return (...args) => {
      dispatch(actionCreators(...args))
    }
  }else if(typeof actionCreators === 'object') {
    const keys = Object.keys(actionCreators);
    let res = {};
    keys.forEach((key) => {
      res[key] =  (...args) => {
        dispatch(actionCreators[key](...args))
      }
    })
    return res;
  }
}
```

再次测试两种情况，没问题。

```js
let defaultState = {
  cash: 200
}

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DESCREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer);
const handlerCash = bindActionCreators(actionCreators, store.dispatch);

handlerCash.add(200);
handlerCash.decrement(1);

const add = bindActionCreators(actionCreators.add, store.dispatch);
add(200);
console.log(store.getState());
```

### 分析`bindActionCreators`

#### 基本结构

我们来看一下源码，源码的基本架构跟我们的差不多，也是判断`actionCreators`的类型：

```js
export default function bindActionCreators(
  actionCreator,
  dispatch
){
    if (typeof actionCreators === 'function') {
        // ...
    }
    
    if (typeof actionCreators !== 'object' || actionCreators === null) {
        /// ...
    }
    
    // ...
}
```

#### `bindActionCreator`处理函数类型

当传入的`actionCreator`是函数的时候：

```js
if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
}
```

```js
function bindActionCreator(
  actionCreator,
  dispatch
) {
  return function (this, ...args) {
    return dispatch(actionCreator.apply(this, args))
  }
}
```

#### 如果是非对象类型，或者对象类型为`null`时

```js
if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error(
      `bindActionCreators expected an object or a function, but instead received: '${kindOf(
        actionCreators
      )}'. ` +
        `Did you write "import ActionCreators from" instead of "import * as ActionCreators from"?`
    )
  }
```

抛出异常。

#### 处理对象类型

```js
const boundActionCreators: ActionCreatorsMapObject = {}
for (const key in actionCreators) {
const actionCreator = actionCreators[key]
// 这里不忘去校验每个key对应的actionCreator时否是个函数
if (typeof actionCreator === 'function') {
  boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
}
}
return boundActionCreators
```

到这里`bindActionCreators`的实现就完成了。

#### 小结

`bindActionCreators`的第一个参数`actionCreators`支持函数也支持对象类型，传出什么类型，就返回什么类型的结果。
## `compose`的源码分析

我听说过这个函数，在面试题中，但是我基本在业务中没遇到过它...一定是我代码写的不够多，业务不够复杂！[Redux中文文档](https://www.redux.org.cn/docs/api/compose.html)上是这么描述`compose`的：

> 从右到左来组合多个函数。
>
> 当需要把多个 [store 增强器](https://www.redux.org.cn/docs/Glossary.html#store-enhancer) 依次执行的时候，需要用到它。

从右到左，即`compose(funcA, funcB, funcC)`  => `compose(funcA(funcB(funcC())))`。文档上还举了一个栗子：

```js
import { createStore, combineReducers, applyMiddleware, compose } from 'redux'
import thunk from 'redux-thunk'
import DevTools from './containers/DevTools'
import reducer from '../reducers/index'

const store = createStore(
  reducer,
  compose(
    applyMiddleware(thunk),
    DevTools.instrument()
  )
)
```

由于没接触过，这次就直接看源码分析它吧。

```js
export default function compose(...funcs){
    // 如果0个函数，直接返回arg
    if(funcs.length === 0) return arg => arg
    // 只有一个函数，直接返回该函数
    if(funcs.length === 1) return funcs[0]
    // 多个函数，从右到左依次执行（最先执行的是里面的那个函数）
    return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

#### 小结

`compose`的作用是将多个函数链接起来，将一个函数的返回值作为另一个函数的传参进行计算，最后得出最终的返回值。它的特点是从右到左执行，内部实现的过程中使用了`reduce`方法。

## `applyMiddleware`的源码分析

#### 使用`applyMiddleware`

[Redux 中文文档](https://www.redux.org.cn/)中介绍到`applyMiddleware`接受的参数为`...middlewares`:

> `...middlewares` (*arguments*): 遵循 Redux *middleware API* 的函数。每个 middleware 接受 [`Store`](https://www.redux.org.cn/docs/api/Store.html) 的 [`dispatch`](https://www.redux.org.cn/docs/api/Store.html#dispatch) 和 [`getState`](https://www.redux.org.cn/docs/api/Store.html#getState) 函数作为命名参数，并返回一个函数。该函数会被传入 被称为 `next` 的下一个 middleware 的 dispatch 方法，并返回一个接收 action 的新函数，这个函数可以直接调用 `next(action)`，或者在其他需要的时刻调用，甚至根本不去调用它。调用链中最后一个 middleware 会接受真实的 store 的 [`dispatch`](https://www.redux.org.cn/docs/api/Store.html#dispatch) 方法作为 `next` 参数，并借此结束调用链。所以，middleware 的函数签名是 `({ getState, dispatch }) => next => action`。

这里我们借鉴一下[Redux 中文文档](https://www.redux.org.cn/)中的栗子,开发一个`logger`的中间件:

```js
const logger = ({getState}) => {
  return (next) => (action) => {
    console.log('will dispatch', action);
    // 调用 middleware 链中下一个 middleware 的 dispatch。
    let returnValue = next(action)
    console.log('state after dispatch', getState())
    return returnValue
  }
}
```

```js
let defaultState = {
  cash: 200
}

const reducer = (state = defaultState, action) => {
  const { type, data } = action;
  
  switch(type){
    case INCREMENT: return { ...state, cash: state.cash + data };
    case DESCREMENT: return { ...state, cash: state.cash - data };
    default: return state;
  }
}

let store = createStore(reducer, applyMiddleware(logger));

store.dispatch(add(100));

// will dispatch {type: "INCREMENT", data: 100}
// state after dispatch {cash: 300}
```

这里没猜出来内部的实现，只能大概猜出：`applyMiddleware`支持多个中间件，每个中间件执行的时候，`applyMiddleware`需要将`{dispatch, getState}`给中间件函数，类似于：

```js
middleware({dispatch, getState})
```

然而没什么帮助，我直接去看源码了。

#### 分析`applyMiddleware`

#### 基础架构

在`createStore`源码中,有这么一行代码:

```js
// 当传入了增强器时
if (typeof enhancer !== 'undefined') {
    // 先要保证增强器是一个函数
    if (typeof enhancer !== 'function') {
      throw new Error(
        `Expected the enhancer to be a function. Instead, received: '${kindOf(
          enhancer
        )}'`
      )
    }

    // 调用增强器
    return enhancer(createStore)(
      reducer,
      preloadedState
    )
}
```

`enhancer`就是`createStore`中的第三个参数，即`applyMiddleware`最后返回的结果，从这里可以分析出`applyMiddleware`接受参数`createStore`然后返回一个新的函数，这个函数接受：`reducer`和`preloadeState`两个参数。

`createStore`最后返回的是一个`store`对象，那么如果有`enhancer`，即使用了`applyMiddleware`，那么最后`return enhancer`最终结果应该也是返回一个`store`对象，不同的是`dispatch`被特殊处理了一下。

```js
export default function applyMiddleware(...middlewares){
    return (createStore) => (reducer, preloadedState) => {
        const store = create(reducer, preloadedState);
        // ...
        return store;
    }
}
```

#### 特殊处理`dispatch`

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
      dispatch = compose(...chain)(store.dispatch)
	  // 将新的dispatch替换原先的store.dispatch
      return {
        ...store,
        dispatch
      }
    }
}
```

看完我还是有点云里雾里，于是我下一个源码分析，我要看看`redux-thunk`，再来对着理解一波。

## 总结

`redux`的源码前面的部分还算简单，最后的`applyMiddleware`有点懵逼。这次读源码最大的收获是边界性的考虑吧，里面很多都做了类型判断，这值得我去学习。还学了一堆可以用到业务中的逻辑，比如如何判断一个简单对象、`Number.prototype.toString(radix)`、`compose`等。


## 参考

- [redux1 - 20行代码实现redux](https://blog.csdn.net/qq_36407748/article/details/106382734)
- [我的源码阅读之路：redux源码剖析](https://segmentfault.com/a/1190000016460366)
- [Redux 中文文档](https://www.redux.org.cn/)

