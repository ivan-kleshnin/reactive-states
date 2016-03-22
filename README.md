# Reactive state implementations

**Reactive state (reducer) implementations discussed and compared.**

There are a bunch of approaches actually and I'm surprised noone bothered to make such list so far.

RxJS is used for code samples but everything is supposed to be tech agnostic.

## Basic Reducer pattern

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

Now what if you want to reset state to some initial (seed) value? You can't obviously `update.onNext(0)` because it basically means `s + 0`. The best thing you can do is `update.onNext((+/-)currentState)` and for more complex states it turns to become practically impossible. 

### Benefits 

* the simplest one

### Drawbacks

* fails to describe more complex action sets
* does not support nested state

### Conclusion

* is perfect for cases it satisfies ;)

## Action Reducer pattern

### Elm style

Requires static types. See Redux style for closest analogy.

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
* supports nested state

### Drawbacks

* can't extend reducer you don't control (need to create new one and compose them i.e. expression problem)
* reducer contains a list of possible actions (fast growing if / switch statement i.e. expression problem)
* incidental complexity (middlemen) (data structure kinda conveys a list of possible actions already)
* need to edit multiple files to implement one action

### Conclusion

* Elm / Redux / "Paqmind" styles are equivalent
* are good for basic-to-medium action sets
 
## Functional Reducer pattern

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
* pairs nicely with currying

### Drawbacks

* open action set (can be viewed as a drabwack as well)

### Conclusion

* is good for any action set (maybe an overkill for simplest ones...)
* some solution for nested state is required (see below)

---

Now neither of patterns structurally predefines the number of stores you should have. 
There can be a convention inside a community but still not a structural requirement. 

Additional stores can allow you to separate subscriptions thus avoiding exsessive recalculations (like widget redrawing every time *something irrelevant* was changed in the state...). At the same time additional stores are harder to serialize / track history for etc. So it seems an open engineering question and app specifics should drive your choice. And frameworks should enforce experimenting with that rather than prodive dogmatic "best practices" (no exact framework implied – just a comment).
