---
title: redux-源码解析(1)-createStore
date: 2018-07-25 22:31:41
tags:
---

## createStore

顾名思义,用函数可以创建一个`store`对象,`store`负责保存和管理`state`,利用闭包特性,外部无法直接修改`state`,只能通过`store.getState()`来读取`state`,我们首先看看`createStore`函数的函数签名

```js
createStore(reducer, [preloadedState], [enhancer])
```
在`creatStore`函数前几行有一段代码,实现了`createStore`函数的重载,所以`createStore`函数还可以这么调用


```js
createStore(reducer, enhancer){
  return store = {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  }
}
```

`createStore`返回一个`stre`对象,我们来看看`createStore`的内部结构,简单地说明下部分代码

```js
createStore(reducer, preloadedState, enhancer) {
  // 实现函数重载
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }
    
  // 校验enhancer,对store进行增强,本篇文章暂时不做分析
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    return enhancer(createStore)(reducer, preloadedState)
  }
    
  // 校验reducer,reducer必须是个纯函数,reducer后面会有单独说明
  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }
    
  // 保存相关数据
  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false // reducer是否执行中
  
  // 若nextListeners和currentListeners全等,给nextListeners赋值currentListeners的浅拷贝
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }
   
  // 读取store的state.
  function getState() {
    // 注意!!!在reducer执行过程中无法使用该函数去读取state
    if (isDispatching) {
      throw new Error(
        '在reducer执行中不能通过getState读取state,'+
        '从reducer中的state参数中读取,而不是通过getState'
      )
    }
    return currentState
  }
  
  // 添加订阅者,参数是个函数.注意!!在reducer执行过程中无法调用该函数
  function subscribe(listener) {/*后面会详细说明*/}
    
  // 发送atcion
  function dispatch(action) {/*后面会详细说明*/}
    
  // 替换reducer
  function replaceReducer(nextReducer) {
    // 校验参数类型,必须为函数
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
	
    currentReducer = nextReducer
    // 替换完成后,发送REPLACE的action
    dispatch({ type: ActionTypes.REPLACE })
  }
    
  // 创建store的最后一步
  // 发送INIT的action
  // 在reducer中不该对type为ActionTypes.INIT的action经行精确命中
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  }
}
```



## reducer

`reducer`是一个纯函数,它是唯一能修改`store`状态的途径,当`store`发送一个`action`时`reducer`会在`store`内部被执行,它的函数签名是

```js
// state: 当前store的state
// action: 外部代码执行store.dispatch传入的action
reducer(state, action){
    /* 通过对action.type的判断,执行相应代码,返回一个新的state */
    /* reducer是一个纯函数,所以不要对state进行修改 */
    return newState
}
```
>  `reducer`内部怎么写都行,只要是个纯函数,返回一个新的`state`就可以,这里建议引入`immutable`

我们来看一个简单的reducer例子, 对state进行加减

```js
import {__DO_NOT_USE__ActionTypes as ActionTypes} from 'redux'

const reducer = (state = 0, action) => {
  switch (action.type) {
    case 'increment':
      return state + 1;
    case 'decrement':
      return state - 1
    case ActionTypes.INIT: 
    // 注意!!!!
    // 该case是个错误的示例,不能直接命中 ActionTypes.INIT
    // 从redux的命名可以看,不应在redux的外部使用该值
      return state
    default:
      return state
  }
}
```

## subscribe

`subscribe`用来添加订阅,当`store`发送一个`action`时,被订阅的函数就会执行.

```js
function subscribe(listener) {
  // 参数校验
  if (typeof listener !== 'function') {
    throw new Error('Expected the listener to be a function.')
  }
  // reducer执行中无法添加订阅
  if (isDispatching) {
    throw new Error(
      'reducer执行中无法添加订阅'
    )
  }
  // 是否已订阅的标识
  let isSubscribed = true
  ensureCanMutateNextListeners()
  // 在订阅队列中push一个listener
  nextListeners.push(listener)
  // 返回该次订阅的取消订阅函数
  return function unsubscribe() {
    // 执行过取消订阅的函数后,再次调用,后面的代码就不需要执行
    if (!isSubscribed) {
      return
    }
    // reducer执行中无法取消订阅
    if (isDispatching) {
      throw new Error(
        'reducer执行中无法取消订阅'
      )
    }
    // 标记取消订阅
    isSubscribed = false

    ensureCanMutateNextListeners()
    // 从订阅队列中删除订阅的事件
    const index = nextListeners.indexOf(listener)
    nextListeners.splice(index, 1)
  }
}
```

> 在发送action后,reducer执行完毕,会依次按添加订阅的**顺序**执行`listeners`

## dispatch

`dispatch`用来发送action,在`dispatch`内部执行`reducer`,执行返回的结果就是store的最新`state`, 然后**按顺序**执行`listeners`

```js
 function dispatch(action) {
    // action格式的校验
    if (!isPlainObject(action)) {
      throw new Error('action的格式必须是简单的对象,用中间件处理异步的action')
    }
    
    if (typeof action.type === 'undefined') {
      throw new Error('action必须要有type属性')
    }
	
    if (isDispatching) {
      throw new Error('有reducer未执行完毕,无法发送action')
    }

    try {
      isDispatching = true
      // 执行reducer,执行完毕后赋值赋值给state,更新状态
      currentState = currentReducer(currentState, action)
      // 还记得前面reducer的例子吗,回头看看,currentReducer就是那个函数
    } finally {
      isDispatching = false
    }
     
    const listeners = (currentListeners = nextListeners)
    // 循环执行订阅的事件
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
    // 最后返回action
    return action
  }
```

## 总结

光从`redux`的`creatStore`函数的代码可以看出整个`redux`的基本运行机制类似发布订阅模式, `reducer`就是订阅的所有事件,  `actio.type`是事件的类型, `dispatch`一个`action`就是广播一个事件.区别在于`redux`无法直接添加事件.但可以通过`replaceReducer`函数替换`reducer`. 