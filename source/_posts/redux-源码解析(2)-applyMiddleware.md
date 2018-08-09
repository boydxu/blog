---
title: redux-源码解析(2)-applyMiddleware
date: 2018-08-09 21:29:49
tags:
---

讲到redux中间件就涉及好多知识了, 如函数的柯里化、`store`的强化器`enhancer`、`compose`归并方法,这里不讲函数的柯里化.

## enhancer

> `enhancer`是一个高阶函数.顾名思义,它用来增加或者修改store的功能, 在`createStore`中只有短短的几行代码涉及到它.


先来看看它的函数签名,它接收`createStore`函数,返回一个接收`reducer`和`preloadedState`的函数

```js
function enhancer(createStore) {
  
  return(reducer, preloadedState) => {
    // ...
  	// 创建一个store
   	// 对store进行修改
  	// ...
    // 返回store
  }
}
```

看到这里,你需要知道它大概的样子就好.

来个实际的场景,我们需要对每次发送的`action`进行输出log.在理解`enhancer`前,你可以会这么处理,在每个`reducer`中添加代码,对`action`输出

```js
const reducer = (state = 0, action) => {
  console.log(action)
  switch (action.type) {
    case 'increment':
      return state + 1;
    case 'decrement':
      return state - 1
    default:
      return state
  }
}
```

简单粗暴的解决方法,但如果`reducer`有很多个,那么我们需要对所有的`reducer`添加这段代码,容易遗漏, 当我们不需要这段输出的时候,那么我们又要从多个`reducer`中删除这段代码,给后期的维护提高了难度.

这个时候该`enhancer`出场了,`reducer`是在`dispatch`中被调用的,那么我们修改`dispatch`函数就完美地解决了这个问题,来看看代码.

```js
const enhancer = (createStore) => {
  return (reducer, preloadedState) => {
    // 使用原createStore创建store
    const store = createStore(reducer, preloadedState)
    // 保存原dispatch的引用
    let dispatch = store.dispatch
    // 修改store的dispatch方法
    store.dispatch = (action) => {
      console.log(action);
      // 调用原dispatch方法
      dispatch(action)
    }
    return store
  }
}
```

> 思考:如果继续增加逻辑,计算每次dispatch执行的时间以及输出dispatch执行前的state和dispatch执行后的state.

继续看代码

```js
const enhancer = (createStore) => {
  return (reducer, preloadedState) => {
    const store = createStore(reducer, preloadedState)
    let dispatch = store.dispatch
    store.dispatch = (action) => {
      console.log(action)
      console.log(store.getState())
      console.log(new Date())
      dispatch(action)
      console.log(new Date())
      console.log(store.getState())
    }
    return store
  }
}
```

> 还好我们只是增加了一些简单的代码,假设如果是一些很复杂的代码,各种逻辑的代码混杂的一起,后期代码维护工作的难度还是不小的.
>
> 同时思考如果其中部分的代码需要给其他项目复用呢?总不能老是copy代码吧.这时候该中间件出场了

## middleware中间件

每个`middleware`是个独立的高阶函数. `middleware`中间件简单的理解就是在`dispatch(action)`前后添加自己的代码.middleware类似洋葱圈模型. 

![Snipaste_2018-08-08_18-08-31](/images/Snipaste_2018-08-08_18-08-31.png)


通过图可以看出,store.dispatch(经过封装的dispatch)后,代码执行路径是mid1 -> mid2 -> mid3 -> dispatch(原store.dispatch) -> mid3 -> mid2 -> mid1.


## applyMiddleware

`applyMiddleware`函数的本质是创建一个`enhancer`,它传入多个中间件函数,在返回的`enhancer`中一层层地将原store.dispatch嵌套.

先看看中间件的函数签名,函数签名比较复杂,内部有2层的函数嵌套.

```js
// 按照层次,依次参数为
// 1. store对象,
// 2.下一个中间件的最内层函数或者是原store.dispatch,
// 3. 上一层中间件返回的action对象,或者是最外层调用store.dispatch函数的参数action
const middleware = store => next => action => {
}
```

来看看上面讲到的三个需求用中间件的实现代码.

```js
// 在执行dispatch前输出action
const actionLogMiddleware = store => next => action => {
  console.log(action)
  let r = next(action)
  return r
}

// 在执行dispatch前后输出state
const stateLogMiddleware = store => next => action => {
  console.log(store.getState())
  let r = next(action)
  console.log(store.getState())
  return r
}

// 输出dispatch的执行时间
const timeLogMiddleware = store => next => action => {
  console.log('timeStart',new Date())
  let r = next(action)
  console.log('timeEnd',new Date())
  return r
}
const reducer = (state = 0, action) => {
  console.log('exec reducer')
  if(action.type === 'add'){
    return state + 1
  }
  return state
}
// 我刚才说过applyMiddleware函数的本质是创建一个enhancer
const enhancer = applyMiddleware(actionLogMiddleware, stateLogMiddleware, timeLogMiddleware)
let store = createStore(reducer, enhancer)
store.dispatch({ type: 'add' })
// 各中间件依次输出
// actionLogMiddleware:    { type: 'add' }
// stateLogMiddleware:     0
// timeLogMiddleware:      timeStart xxxx
// reducer:                exec reducer
// timeLogMiddleware: 	   timeEnd xxxx
// stateLogMiddleware: 	   1

```

来看看`applyMiddleware`是怎么实现这点的

```js
function applyMiddleware(...middlewares) { // middlewares = [mid1, mid2, mid3]
  return createStore => (...args) => {
    // 创建原store
    const store = createStore(...args)
    // 在Dispatching中添加中间件会报一下错误,但是在store还未创建,dispatch怎么会执行?这点不太明白
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
        `Other middleware would not be applied to this dispatch.`
      )
    }
    // 定义一个类store对象,包含getState函数和上面我们刚定义的dispatch函数
    // 不过在后面dispatch指向又变了,所以在中间件访问到的store.dispatch函数其实是封装过的dispatch
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    
    // 类store对象作为参数,遍历执行所有的中间件
    // chain是以下结构的函数构成的数组
    // func = (next) => action => { /*在这里可以访问到类store对象*/ }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    
    // compose函数是一个归并方法,对chain做了一次归并操作,compose函数的解析请看下一小节
    // 这段代码是对原dispatch的封装
    // 这里可能不太好理解,先跳过,看完compose解析后再回来看这里
    dispatch = compose(...chain)(store.dispatch)
    
    // 返回原store的拷贝,并将原dispatch函数替换成封装过的dispatch函数
    return {
      ...store,
      dispatch
    }
  }
}
```

## compose

compose是一个归并方法, 作用是将中间件的层层嵌套执行,来看看源码

```js
export default function compose(...funcs) {
  // 没有参数时直接返回 arg => arg 
  if (funcs.length === 0) {
    return arg => arg
  }
  // 只有一个参数时,直接返回参数
  if (funcs.length === 1) {
    return funcs[0]
  }
  // 对参数归并执行,看下一段代码,了解它的作用
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

```js
const func1 = (a) => { /**/ }
const func2 = (a) => { /**/ }
const func3 = (a) => { /**/ }
const chain = [func1, func2, func3]

const func4 = compose(...chain)
// 最后hunc4是这个样子
// func4 = (args) => func1(func2(func3(args)))
```

到这里,你需要模糊地明白compose做了哪些事情就好了,最后拿一个例子做一次关键代码的执行解析

```js
const mid1 = store1 => next1 => action1 => {
  /* do something 1 */
  let r1 = next1(action1)
  /* do something1 1 */
  return r1
}

const mid2 = store2 => next2 => action2 => {
  /* do something 2 */
  let r2 = next2(action2)
  /* do something 2 */
  return r2
}

const mid3 = store3 => next3 => action3 => {
  /* do something 3 */
  let r3 = next3(action3)
  /* do something 3 */
  return r3
}
//  执行applyMiddleware(mid1,mid2)
applyMiddleware(mid1, mid2, mid3){
  const middlewares = [mid1, mid2, mid3]
  return createStore => (reducer, preloadedState)) => {
    const store = createStore(reducer, preloadedState)
    let dispatch = () => { throw new Error('xxxx') }
    // 创建类store对象
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }

    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 得到的chain为以下结构,但map遍历执行后都生成了一个闭包,
    // 所以每个函数都可以访问到middlewareAPI对象,
    // 注意此时middlewareAPI.dispatch还是那个抛出错误的箭头函数
    chain = [
      next1 => action1 => {
        /* do something 1 */
        let r1 = next1(action1)
        /* do something 1 */
        return r1
      },
      next2 => action2 => {
        /* do something 2 */
        let r2 = next2(action2)
        /* do something 2 */
        return r2
      },
      next3 => action3 => {
        /* do something 3 */
        let r3 = next3(action3)
        /* do something 3 */
        return r3
      }
    ]

    dispatch = compose(...chain)(store.dispatch)
    // compose(...chain)执行结果类似以下结构
    // 注意,每个next函数同样都是通过闭包的形式去访问的,而不是下面代码这样,但整体结构类似
    let preDispatch = (next3) => (action1) => {
      /* do something 1 */
      let next1 = (action2) => {
        /* do something 2 */
        let next2 = (action3) => {
          /* do something 3 */
          let r3 = next3(action3)
          /* do something 3 */
          return r3
        }
        let r2 = next2(action2)
        /* do something 2 */
        return r2
      }
      let r1 = next1(action1)
      /* do something 1 */
      return r1
    }
    
    // 最后传入原sotre.dispatch执行得到封装过的dispatch函数
    // dispatch = preDispatch(sotre.dispatch)
    // 注意,此时dispatch的指针改变了,上面middlewareAPI.dispatch也就指向了封装过的dispatch
    // 所以每个中间件是可以调用封装过的dispatch函数的
    // 最后得到的dispatch是这样的
    dispatch = (action1) => {
      /* do something 1 */
      let next1 = (action2) => {
        /* do something 2 */
        let next2 = (action3) => {
          /* do something 3 */
          // 执行原store.dispatch
          let r3 = store.dispatch(action3)
          /* do something 3 */
          return r3
        }
        let r2 = next2(action2)
        /* do something 2 */
        return r2
      }
      let r1 = next1(action1)
      /* do something 1 */
      return r1
    }

    // 返回原store的拷贝,并将原dispatch函数替换成封装过的dispatch函数
    return {
      ...store,
      dispatch
    }
  }
}
```

到此,applyMiddleware源码解析完成 . 要想更好了解redux,还是推荐自己写一遍简单的用例,一步步调试,输出关键数据才能透彻地了解redux的执行机制.
