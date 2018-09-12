---
title: redux-源码解析(3)-bindActionCreators&combineReducers
date: 2018-09-12 20:41:03
tags: redux
---

bindActionCreators和combineReducers逻辑比较简单,不像之前的applyMiddleware那么难理解

## bindActionCreators

`bindActionCreator`的主要功能是用`dispatch`将`action创建函数`包裹起来,返回的函数执行后会自动dispatch`action创建函数`生成的`action`.

看两个简单🌰

```js
const store = createStore(reducer)

function createIncrementAction(){
    return {
        type: 'increment'
    }
}
// 通过bindActionCreators将createIncrementAction用dispatch包裹
let autoDispatchAction = bindActionCreators(createIncrementAction,store.dispatch)

autoDispatchAction() // 创建action后,执行了dispatch({type:'increment'})

```

当actionCreators是对象的情况

```js
const actionCreators = {
    increment: () => {
       return {type: 'increment'}
    }
    decrement: () => {
       return {type: 'decrement'}
    }
}

let autoDispatchAction = bindActionCreators(actionCreators,store.dispatch)

actionCreators.increment() // 最终执行了dispatch({type:'increment'})
actionCreators.decrement() // 最终执行了dispatch({type:'decrement'})
```



下面我们来看看源码

```js

function bindActionCreator(actionCreator, dispatch) {
  // 返回一个匿名函数, 执行该匿名函数,先执行原action创建函数,再执行dispatch
  return function() {
    return dispatch(actionCreator.apply(this, arguments))
  }
}
// 核心代码,就上面几行

// actionCreators,参数是action创建函数,也可以是属性都是'action创建函数'的对象
export default function bindActionCreators(actionCreators, dispatch) {
  // 在actionCreators是action创建函数的情况下直接调用bindActionCreator
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }
  // 对actionCreators进行校验
  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error(
      `bindActionCreators expected an object or a function, instead received ${
        actionCreators === null ? 'null' : typeof actionCreators
      }. ` +
        `Did you write "import ActionCreators from" instead of "import * as ActionCreators from"?`
    )
  }
  // 读取actionCreators所有属性名
  const keys = Object.keys(actionCreators)
  // 定义结果变量
  const boundActionCreators = {}
  
  // 遍历actionCreators的所有属性值
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    // 如果该属性值是函数的话,给结果变量添加相同属性,赋值bindActionCreator函数的执行结果
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  // 返回结果变量
  return boundActionCreators
}
```



## combineReducers

combineReducers的作用也很简单,对多个reducer进行了合并,源码虽然有100多行,但核心代码也只有60左右,原理也非常简单,老样子,先看个使用🌰

```js
const incrementReducer = (state = 0, action) => {
  switch (action.type) {
    case 'increment':
      return state + 1;
    default:
      return state
  }
}
const decrementReducer = (state = 0, action) => {
  switch (action.type) {
    case 'decrement':
      return state - 1
    default:
      return state
  }
}

let reducers = combineReducers({
  incrementReducer,
  decrementReducer,
  otherReducer: decrementReducer,
})

let store = createStore(reducers)
store.getState()
// 得到下面state
// {
//   incrementReducer: 0,
//   decrementReducer: 0,
//   otherReducer: 0
// }
store.dispatch({
  type: 'increment'
})
store.getState()
// 得到下面state
// {
//   incrementReducer: 0,
//   decrementReducer: -1,
//   otherReducer: -1
// }
```

可以看到combineReducers的参数是一个对象,它的每个属性都是一个reducer.该对象分别有`incrementReducer`,`decrementReducer`,`otherReducer`三个属性,最终获取到的state有相同的属性. 每个reducer只能对各自同名的state属性进行值的修改.  在`dispatch(action)`之后,state有2个属性值发生了改变,因为每个`reducer`都被调用了,但只有下面2个reducer的switch命中了该type,所以只有同名的state属性值发生了改变.

下面看看源码,为了方便解读,我调整了函数顺序.

```js
import ActionTypes from './utils/actionTypes'
import warning from './utils/warning'
import isPlainObject from './utils/isPlainObject'

export default function combineReducers(reducers) {
  // 读取reducers所有属性
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {}
  // 遍历reducers所有属性,下面几行代码纯粹是为了净化掉非function值的属性
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }
    // 属性值为function,给finalReducers添加同名属性并赋值
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  // 读取过滤属性后的reducers(也就是finalReducers)的所有属性
  const finalReducerKeys = Object.keys(finalReducers)

  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }
  // 还是校验reducer,具体见assertReducerShape函数
  let shapeAssertionError
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }
  // combineReducers的执行结果是一个函数,这个函数就是新的reducers,它和普通的reducer一样,只接收state和action参数
  // 注意: 在使用组合reducers后,state的初始值是个空对象,state也可以在createStore时指定,但也必须是对象
  return function combination(state = {}, action) {
    // 在assertReducerShape不通过的情况执行下调用新的recuder时抛出错误
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    if (process.env.NODE_ENV !== 'production') {
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
    // state是否改变的标识
    let hasChanged = false
    // 新的state,所有的reducer执行后的结果都保存在里面
    const nextState = {}
    // 遍历reducers
    for (let i = 0; i < finalReducerKeys.length; i++) {
      // key为该reducer在reducers的属性名
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      // state同属性名的值
      const previousStateForKey = state[key]
      // 执行该reducer,得到新的状态
      const nextStateForKey = reducer(previousStateForKey, action)
      // 执行结果为unundefined,抛出错误
      if (typeof nextStateForKey === 'undefined') {
        // getUndefinedStateErrorMessage对错误信息进行拼接
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      // 把执行结果保存在nextState对应的属性
      nextState[key] = nextStateForKey
      // 判断状态是否发生改变,更新标识
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    // 执行完所有的reducer后,根据state是否改变的标识,返回对应结果.
    return hasChanged ? nextState : state
  }
}
// 以上为核心代码,下面代码可以忽略
// 以上为核心代码,下面代码可以忽略
// 以上为核心代码,下面代码可以忽略


// 对所有的reducer的执行结果做一次检测
// 保证每个reducer执行的结果为非undefined.
// 注意!!执行结果可以为null
function assertReducerShape(reducers) {
  // 遍历每个reducer
  Object.keys(reducers).forEach(key => {
    const reducer = reducers[key]
    // 模拟在creatStore中执行dispatch({type: ActionTypes.INIT}),结果为undefined就抛错
    const initialState = reducer(undefined, {
      type: ActionTypes.INIT
    })

    if (typeof initialState === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined during initialization. ` +
        `If the state passed to the reducer is undefined, you must ` +
        `explicitly return the initial state. The initial state may ` +
        `not be undefined. If you don't want to set a value for this reducer, ` +
        `you can use null instead of undefined.`
      )
    }
    // 模拟在creatStore中执行dispatch,action.type为随机字符串,结果为undefined就抛错
    const type =
      '@@redux/PROBE_UNKNOWN_ACTION_' +
      Math.random()
        .toString(36)
        .substring(7)
        .split('')
        .join('.')
    if (typeof reducer(undefined, {
      type
    }) === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined when probed with a random type. ` +
        `Don't try to handle ${
        ActionTypes.INIT
        } or other actions in "redux/*" ` +
        `namespace. They are considered private. Instead, you must return the ` +
        `current state for any unknown actions, unless it is undefined, ` +
        `in which case you must return the initial state, regardless of the ` +
        `action type. The initial state may not be undefined, but can be null.`
      )
    }
  })

  // 错误信息拼接
  function getUndefinedStateErrorMessage(key, action) {
    const actionType = action && action.type
    const actionDescription =
      (actionType && `action "${String(actionType)}"`) || 'an action'

    return (
      `Given ${actionDescription}, reducer "${key}" returned undefined. ` +
      `To ignore an action, you must explicitly return the previous state. ` +
      `If you want this reducer to hold no value, you can return null instead of undefined.`
    )
  }

  function getUnexpectedStateShapeWarningMessage(
    inputState,
    reducers,
    action,
    unexpectedKeyCache
  ) {
    const reducerKeys = Object.keys(reducers)
    const argumentName =
      action && action.type === ActionTypes.INIT ?
        'preloadedState argument passed to createStore' :
        'previous state received by the reducer'

    if (reducerKeys.length === 0) {
      return (
        'Store does not have a valid reducer. Make sure the argument passed ' +
        'to combineReducers is an object whose values are reducers.'
      )
    }

    if (!isPlainObject(inputState)) {
      return (
        `The ${argumentName} has unexpected type of "` + {}.toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
        `". Expected argument to be an object with the following ` +
        `keys: "${reducerKeys.join('", "')}"`
      )
    }

    const unexpectedKeys = Object.keys(inputState).filter(
      key => !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key]
    )

    unexpectedKeys.forEach(key => {
      unexpectedKeyCache[key] = true
    })

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
}
```

