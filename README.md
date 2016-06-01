# Reactive states

RxJS is used for code samples but everything is supposed to be technology agnostic.

---

Reactive state is a state implemented by reducing values over time. There are two main questions to be asked for every particular library which provides a "state solution". Namely: the exact implementation of reducers and the number of them (one vs few vs many).

This two questions are completely orthogonal yet framework authors does not emphasize this fact enough.
Saying "Redux architecture" the first person may have a "single store" concept in mind. The second one may imply "action-style reducers" while the third may assume a strict combination of both.

Let's start with possible reducer implementations.

## Reducer implementations

### Data reducer

```js
let {Observable, Subject} = require("rx")

let update = new Subject() // update channel

let state = update 
  .startWith(0)          // initial value
  .scan((s, x) => s + x) // data reducer

state.subscribe(s => console.log("state:", s))

// TEST
update.onNext(1)  // +1
update.onNext(2)  // +2
update.onNext(3)  // +3
update.onNext(-3) // -3
update.onNext(-2) // -2
update.onNext(-1) // -1
```

```
state: 0
state: 1
state: 3
state: 6
state: 3
state: 1
state: 0
```

Now what if you want to reset state to some initial (seed) value? Obviously, you can't `update.onNext(0)` because it means `s + 0`. The best thing you can do is `update.onNext((+/-)currentState)` and you can guess how quick this becomes unreasonable.

#### Benefits

* the simplest one

#### Drawbacks

* fails to represent all but the simplest action sets
* fails to represent nested state

#### Conclusion

* is perfect for cases it satisfies ;)

### Action Reducer 

#### Redux way

```js
let {Observable, Subject} = require("rx")

let update = new Subject() // update channel

let ADD = "+"
let SUBTRACT = "*"

let state = update 
  .startWith(0) 
  .scan((state, action) => {
    switch (action.type) {
      case ADD:
        return state + action.value
      case SUBTRACT:
        return state - action.value
      default:
        throw Error("unsupported action")
    }
  })

state.subscribe(s => console.log("state:", s))

// TEST
update.onNext({type: "+", value: 1}) // +1
update.onNext({type: "+", value: 2}) // +2
update.onNext({type: "-", value: 2}) // -2
update.onNext({type: "-", value: 1}) // -1
```

```
state: 0
state: 1
state: 3
state: 1
state: 0
```

#### Alternative way

```js
let {Observable, Subject} = require("rx")

let update = new Subject() // update channel

let state = update 
  .startWith(0)
  .scan((state, action) => {
    switch (action[0]) {
      case "+":
        return state + action[1]
      case "-":
        return state - action[1]
      default:
        throw Error("unsupported action")
    }
  })

state.subscribe(s => console.log("state:", s))

// TEST
update.onNext(["+", 1]) // +1
update.onNext(["+", 2]) // +2
update.onNext(["-", 2]) // -2
update.onNext(["-", 1]) // -1
```

```
state: 0
state: 1
state: 3
state: 1
state: 0
```

Now what if we want to reset counter here? It's just a matter of adding a new switch branch.

#### Benefits

* can describe arbitrarily complex action sets
* reducer contains a list of possible actions
* can represent nested state
* easy to log actions

The most interesting benefit of this one is the ability to log actions easily.
You just need to prepend a reducer with a logger as all you need is there (in the pipe).
This comes in handy for scrapers where you need to log all pages you download, all documents you save, etc.

#### Drawbacks

* can't extend reducer you don't control (need to create new one and compose them: expression problem)
* reducer contains a list of possible actions (fast growing if / switch statement: expression problem)
* incidental complexity (middlemen) (data structure kinda conveys a list of possible actions already)
* need to edit multiple files to implement one action (action & reducer files) 

#### Conclusion

* are good for basic-to-medium action sets (?)
* or if you need a no-brainer for logging

### Functional Reducer 

```js
let {add, curry, flip, subtract} = require("ramda")
let {Observable, Subject} = require("rx")

// scanFn :: s -> (s -> s) -> s
let scanFn = curry((state, updateFn) => {
  if (typeof updateFn != "function" || updateFn.length != 1) {
    throw Error("updateFn must be a function with arity 1, got " + updateFn)
  } else {
    return updateFn(state)
  }
})

let update = new Subject() // update channel

let state = update 
  .startWith(0)
  .scan(scanFn)

state.subscribe(s => console.log("state:", s))

// TEST
update.onNext(add(1))            // +1
update.onNext(add(2))            // +2
update.onNext(add(3))            // +3
update.onNext(flip(subtract)(3)) // -3
update.onNext(flip(subtract)(2)) // -2
update.onNext(flip(subtract)(1)) // -1
```

#### Benefits

* open action set (not limited by hardcoded actions)
* composable actions (`update3 = compose(update2, update1)`)

#### Drawbacks

* open action set (can be viewed as drawback)
* hard to log actions (stream of functions is obscure)

#### Conclusion

* pairs nicely with currying
* pairs nicely with lensing
* is appropriate for most action sets out there (?)

## Derived states

What is **derived state**? It's a state which is produced from other state (common or derived as well).<br/>
"So why is this not just a state? Or just a stream?" – you can ask. "Why we need additional terms besides MVC, MVI and whatnot?"
Unfortunately, world is too complex for acronyms.

Form errors is an example of **derived state**. It's a state because it's renderable.
You render input errors in the same way as input values. But there are several reasons to treat them differently.

1. Minimal state size is desirable for serialization (transfer, etc.)
2. Keep one source of truth (avoid unsync cases like `y == f(x) /* by formula */ y != f(x) /* by fact */`.
3. Preserve reactivity (derivables are either passive or reactive).

This should detract you from the idea of mixing common and derived states in a single reducer.
What options are left? 

1) Derived state is a stream (not usual but stateful one). 

```js
let derived = state.map((s) => {
  let d = ...
  // calculate every dependency
  return d
})
```
2) Derived state is a record of streams.

```js
let derived = {
  d1: state.map((s) => ...),
  d2: state.map((s) => ...),
}
```

In a single-stream version, `state.counter` increasing every second will trigger a recalculation of derived state every second.
In a multi-stream version you can describe reactive dependencies per-field `someFlag.debounce(10)` but passing stuff between functions is encomplicated. You can also mix-n-match approaches:

```js
let derived = {          // RECORDS OF STREAMS
  foo: fooStream,        // single-value stream
  bar: barStream,        // ...
  flags: state.map(...), // multi-value stream
}
```


