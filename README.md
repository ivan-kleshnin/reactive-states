# Reactive state implementations 

**A beta release** :turtle: 

**Reactive state APIs and implementations discussed and compared.**

RxJS is used for code samples but everything is supposed to be technology agnostic.

---

Reactive state is a state implemented by reducing values over time. There are two main questions to be asked for every particular library which "provides a state solution". Namely the exact implementation of reducer and the number of reducers.

This two questions are completely orthogonal yet framework authors does emphasize this fact for some reason.
Talking about "Redux architecture" the first person may have a "single store" concept in mind. The second one may imply "action-style reducers" while the third may assume a strict combination of both.

Let's start with possible "reducer" architectures.

## Basic Reducer 

```js
let {add} = require("ramda")
let {Observable, Subject} = require("rx")

let seed = 0 // initial value

let update = new Subject() // update channel

let state = update // basic reducer
  .startWith(seed)
  .scan(add)

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

Now what if you want to reset state to some initial (seed) value? Obviously, you can't `update.onNext(0)` because it means `s + 0`. The best thing you can do is `update.onNext((+/-)currentState)` and you can guess how quick this will become unreasonable.

### Benefits

* the simplest one

### Drawbacks

* fails to represent all but the simplest action sets
* fails to represent nested state

### Conclusion

* is perfect for cases it satisfies ;)

## Action Reducer 

### Elm style

Requires static types. See Redux style for the closest analogy.

### Redux style

```js
let {add} = require("ramda")
let {Observable, Subject} = require("rx")

let seed = 0 // initial value

let update = new Subject() // update channel

let ADD = "+"
let SUBTRACT = "*"

let state = update // action reducer
  .startWith(seed)
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

### "Paqmind" style

```js
let {add} = require("ramda")
let {Observable, Subject} = require("rx")

let seed = 0 // initial value

let update = new Subject() // update channel

let state = update // action reducer
  .startWith(seed)
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

### Benefits

* can describe arbitrarily complex action sets
* reducer contains a list of possible actions
* can represent nested state
* easy to log actions

The most interesting benefit of this one is the ability to log actions easily.
You just need to prepend a reducer with a logger as all you need will be there (in the pipe).
This comes in handy for scapers where you need to log all pages you download, all documents you save, etc.

### Drawbacks

* can't extend reducer you don't control (need to create new one and compose them i.e. expression problem)
* reducer contains a list of possible actions (fast growing if / switch statement i.e. expression problem)
* incidental complexity (middlemen) (data structure kinda conveys a list of possible actions already)
* need to edit multiple files to implement one action
* ability so split / combine reducers requires sophisticated solutions

Talking about the latter. **Transducers** are a one possible solution. Yet they are 1) not well-known in JS community 2) far from being that cool as Clojure guys impose. 

*(Look into the [Ramda](https://github.com/ramda/ramda/search?utf8=%E2%9C%93&q=transducer&type=Code) (or Clojure) codebases to get a proper impression of their "viral" protocol). A mess.*

[Ad-hoc helpers](https://github.com/reactjs/redux/blob/master/src/combineReducers.js) are a second possibility.

Both make me cry.

### Conclusion

* are good for basic-to-medium action sets (?)
* or if you need no-brainer solution for logging

## Functional Reducer 

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

let seed = 0 // initial value

let update = new Subject() // update channel

let state = update // action reducer
  .startWith(seed)
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

### Benefits

* open action set
* natural composition (!)
 
Update functions are all of type `s -> s` so are naturally composable.<br/> 
Nasty `combineReducers` workarounds are not required.

### Drawbacks

* open action set (can be also viewed as drawback)
* hard to log actions (stream of functions is obscure)

### Conclusion

* pairs nicely with currying
* pairs nicely with lensing
* is appropriate for most action sets out there (?)

## Complex state

### Approaches

*It's an open question whether library can contain or predefine a state shape.<br/>
Maybe all such libraries are rather frameworks or co-apps if you will.*

Anyway, there are two fundamentally opposite approaches to represent **reactive state** on the app level.

1. **1-stream state**. Keep all state together.<br/> 
   Popularized by ClojureScript [Om](https://github.com/omcljs/om) library. In JS is used in [Baobab](https://github.com/Yomguithereal/baobab) library. [Redux](https://github.com/reactjs/redux) promotes this approach. 

2. **n-stream state**. Keep state separated between reducers (also known as "atoms").<br/> 
  Popularized by ClojureScript [Reagent](https://github.com/reagent-project/reagent) project. [calmm-js.github.io](http://calmm-js.github.io) promotes this approach.
   
Both styles are perfectly valid but issues they hit are the opposite.
Keeping state together is beneficial for *union* operations. History tracking, transactions, joins, serialize / deserialize questions... whatever requires several reducers becomes easier. The problems emerge when
you want to subscribe on a state fragment. 

Naive implementations where you stuff all the app state into a single stream fail shortly.
You don't want to diff DOM every time state change if that state is not directly visible.
It seems both Redux and Elm share this problem (correct me if I wrong).

More sophisticated implementations allow you to subsribe on a cursor.
Both Om and Baobab are quite complex libraries and it's an interesting question 
whether this state split can be easily implemented.

If you keep state separated (as some collection of streams) the splitting part is already solved.
You need to address joining part. The simplest solution is just to [combine reducers downstream](https://github.com/ivan-kleshnin/cyclejs-examples/blob/master/1.5-form/src/rx.utils.js#L40-L54).
This hovewer can lead to a state being kept in memory two (or even more) times (in different forms).

Still if we can satisfy [immutable databases](https://github.com/tonsky/datascript) keeping state several times
does not sound a game breaker.

### Functional Reducers and State Loops

**n-stream state** being used with **Functional Reducer** naturally produces circular dependencies 
between actions and state streams. This is better be shown by example.

Let's start with a **1-stream state** approach implemented with **Action Reducer** (as Elm / Redux do)
where this problem does not exist.

```js
let actions = [
  Observable.of({type: ADD, value: {name: "Jack"}}),
  Observable.of({type: RESET}),
]
let state = Observable.merge(...actions).
  .startWith({type: RESET})
  .scan(...)
  ...
```

Note that there is no need to have state for performing a `RESET`. As all state is incapsulated in one place it "just works".

For **n** &times; **Functional Reducers** things are getting tricky.

```js
let seeds = {
  users: [] // initial value
}

let actions = [
  Observable.of(append(user)),        // user is available here
  Observable.of(always(seeds.users)), // seeds are available here
]
let state = Observable.merge(...actions).
  .startWith(seeds.users)
  .scan(...)
  ...
```

So far so good. When you start to describe how to register new user declaratively you meet a problem.

```js
let actions = {
  users: {
    create: state // but state is unavailable here... ^_^
      .sample(intents.form.register)
      .map((state) => state.form.output) 
      .filter(identity) 
  },
}

let seeds = ...

let updates = {
  users: {
    data: Observable.merge(
      actions.users.create.map((user) => append(user))
    ),
  },
}

// another state... ^_^ (syntax error)
let state = {
  users: {
    data: store(seeds.users.data, updates.users.data),
  },

  form: {
    input: ...  // raw form data (user input)
    errors: ... // form errors
    output: ... // real model to create (`null` until data is full-n-valid)
  },
}
```

That's because you have a circular [reactive dependency](https://github.com/ivan-kleshnin/dataflows).

```
intents <- actions <- updates <- state
state   <-/
```

See the state in both left and right sides? That's it.

To solve this you need to perform the same trick CycleJS does for [User-Computer interaction](https://www.youtube.com/watch?v=1zj7M1LnJV4).
You'll utilize **laziness** to break-n-join the loop.

```js
function main({state: stateSource}) {
  ...
  let actions = ... // state (named `stateSource`) is available here
  ...
  let stateSink = ... // `updates` are available here (as before)

  return {
    state: stateSink // pass state to the wormhole
  }
}

Cycle.run({
  // loop state
  state: {
    users: {
      data: identity,
    },

    form: {
      input: identity,
      errors: identity,
      output: identity,
    },
  },
})
```

CycleJS describes drivers as something which performs side-effects but this trick
is perfectly valid and working. State pipes are just noop "drivers".
No data is duplicated by looping so we shouldn't create a memory leak (proof?). 

Refer to [working example](https://github.com/ivan-kleshnin/cyclejs-examples/blob/master/x.x-form/src/app.js) of this architecture if you need more details to get all the above.


