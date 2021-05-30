# ES Actors

> Note: precise syntax and naming here is extremely bikesheddable. That's not the point of this proposal.

So, here's the state of state management:

- State management is *difficult*.
- There's [3 billion patterns for it](https://en.wikipedia.org/wiki/Architectural_pattern), virtually none of which are both scalable, flexible, and at least reasonably concise.
    - Model-View-Controller (ex: Backbone, Rails): you either stick to a rigid and rather inflexible near one-to-one relationship between either views and controllers (common on front end) or models and controllers (common on backend) or you lose scalability. (Found this out the hard way.) Also, it's not particularly concise.
    - Model-View-ViewModel (ex: Angular v1, Aurelia, Ember): Same story as MVC, but a little lower level.
    - Model-View-Intent (ex: React + Redux, Cycle.js, Android): it's scalable and moderately flexible, but very prone to boilerplate, and the flexibility falls apart where you define your reducers or intent consumers relative to (views using `connect` in traditional Redux or models in traditional MVI).
    - Component-driven (ex: vanilla React, Vue, Mithril): it's fairly concise (if not done poorly like React) and scales well, but you're forced into this highly rigid and inflexible paradigm. Mithril is very close to the limits of its flexibility.
    - Entity-Component-System (ex: Unity): it's extremely flexible and scales vertically really well, but it gets bloated quick and can run into performance issues after a point.
    - Data-driven (ex: Three.js, date-fns, native Intl, most math libraries, Lua, Clojure): it's very scalable in both performance and code base size and it's especially flexible with monomorphic types, but it gets rigid quick with more abstract data and has a habit of resulting in a decent amount of low-level code when not interfacing through DSL.
    - State machines and related automata (most network code and parsers internally, few libraries actually in use): it's very flexible and can be exceptionally fast, but not the most concise and it scales terribly code-wise. Thanks to math, states tend to multiply quickly, and so all but the smallest state-driven parsers can be quite difficult to reason about, especially for those unfamiliar with computer science.
    - Event-driven (ex: the DOM, native Windows, Java Swing, Node): it's very flexible, but so low level that you basically have to adapt it to a different architecture just to scale it.
    - Actor model (ex: Akka, Play, Elixir): it scales exceptionally well, is flexible, and when done right can be rather concise. And that's where this proposal draws great inspiration.
    - Traditional reactive programming (ex: React Hooks, ReactiveX, Elm): it scales well and is relatively concise for simple streams, but the moment branches get involved in what you're listening to and you need to split outputs, it becomes really unidiomatic really quick and can easily get rather boilerplatey. (I know this one from personal experience.)
- Nearly every web framework and state management library all have many of the same algorithms and concepts.
    - Composition (varies with state management libraries, usually internal in web frameworks)
    - Initialize (usually userland in state management libraries, internal with userland hook in web frameworks)
    - Hydrate (usually internal with userland hook)
    - Destroy/Remove (usually internal with userland hook)
    - Update (usually userland)
    - Commit (usually userland in state management libraries, internal in web frameworks)
    - Fetch (usually userland, but React Suspense and Apollo are notable exceptions)
    - List append (usually userland in state management libraries, internal in web frameworks)
    - List insert (usually userland in state management libraries, internal in web frameworks)
    - List move (usually userland in state management libraries, internal in web frameworks)
    - List remove (usually userland in state management libraries, internal in web frameworks)
    - React to event (varies with state management libraries, usually internal with userland hooks in web frameworks)
    - Delegate event (varies with state management libraries, usually internal in web frameworks)
    - Serialize (usually userland - state management libraries serialize to JSON, web frameworks to either a template invocation or virtual DOM node)

Yeah, let's use a model that's actually proven itself well, and let's learn lessons from what worked well (React Hooks' state mechanism, virtual DOM, actors) and what didn't (MVC, overuse of Redux, Android's intent system). [Time for a new standard!](https://xkcd.com/927/)

## Proposal

```js
// Define an actor function
actor function foo(...args) {
    // ...
}
const foo = actor (...args) => { ... }
class C {
    actor foo() {}
}

// Define an async actor function.
async actor function foo(...args) {
    // ...
}
const foo = async actor (...args) => { ... }
class C {
    async actor foo() {}
}

// Define an actor function that returns generators.
actor function *foo(...args) {
    // ...
}
class C {
    actor *foo() {}
}

// Initialize static variable in actor
// - Run on first invocation always
// - Run if any referenced static variable updates and is not `SameValue` its
//   previous value.
// - Cycles just use whatever the previous value is - it treats this like a
//   single simple shared variable context.
// - Variables outside the immediate actor's scope do not cause the inner actor
//   to update on change.
// - Immediate updates during serialization do *not* trigger additional updates
//   as it's considered redundant at best.
static let foo = bar

// Execute static block - same rules apply as for the expression for static
// variable initializations
static {
    // ...
}

// Manually emit an actor update
static.update()

// Recompute an expression when certain values change via `SameValue`
// - In static contexts, it checks on every actor update
// - In non-static contexts, it checks on every render
static let foo = static.link(...values, expr)
let foo = static.link(...values, expr)

// Ignore a number of bindings when linking dependencies for `expr`. Only usable
// in static contexts.
static let foo = static.ignore(...names, expr)

// Observe invocations to a trap, including receiving arguments for it.
on trap(...args) {
    // ...
}

// Like above, but async
async on trap(...args) {
    // ...
}
async on [dynamicTrapKey](...args) {
    // ...
}

// Schedule a block for closure
do finally {
    // ...
}

// Use a child actor. Returns its render output, and usable in all contexts.
// `actor` and its arguments are all in a static context. The expression is a
// call-like expression - `use expr` is not a valid production.
//
// Traps are proxied through this boundary, as is closure. It's a primitive and
// cannot be implemented in userland.
let result = use actor(...args)
```

- `await` can be used in `async` actor functions in all contexts, and static values' updates are awaited before any non-static value starts. (They're computationally hoisted, even though their references are not.)
- `yield` can be used in generator actor functions (sync or async) in non-static contexts
- On update, all statements execute in source order. `static` statements whose referenced actor-local variables are all `SameValue` their previous values are skipped.
- Variable arguments have their arguments lists compared rather than themselves being individually compared when spread. This only works when the spread name is the same as the parameter name.

### Actor API

- `actorInst = actorFunction(...args)` - Create an actor instance, implicitly sets arguments and runs first update
- `actorInst.update(...args)` - Update the arguments of the actor instance
- `actorInst.send.name(...args)` - Invoke a trap on an actor instance with relevant arguments
    - If an error occurs, it's rethrown
    - May return a promise if any `async on` blocks exist.
    - If the trap has no observers, the method simply does not exist.
- `result = actorInst.render()` - Render an actor instance to its result (the result of `return`)
    - Returns a promise for async actors
- `actorInst.close(ignoreFinally = false)` - Close the actor instance
    - `ignoreFinally` is necessary for certain forms of error handling.
- `subscription = actorInst.subscribe({onUpdate(): void, onError(error): void})` - Observe actor instance updates and actor instance update errors.
    - This does *not* observe renders. It only observes actor instance state changes.
    - These are scheduled via promise jobs. They are not invoked synchronously, to help with optimizability and avoid some very arcane edge cases around update timings.
- `subscription.unsubscribe()` - Close a subscription created per above.

### Bikeshed potential

- Syntax: it's obviously a little ugly and could be refined a bit. Not the point of this proposal, though - I'm shooting more for a stage 1 (where that doesn't matter as much).
- Naming: see above.

### Why make this a language primitive?

- Engines can optimize the slots better, and in some cases elide their existence entirely. (For one, it could use its static knowledge of dependents to simply read a value, write the new one, and compare the old and new values to see if it needs to trigger updates.)
- State updates could avoid 99% of the concerns. An implementation could leverage topological sorting (it's already mostly naturally topologically sorted in general) to let them also make static update entry a single jump to update everything.
- Engines could inline entire actors where useful.
- As explained earlier, it's not specific to UI frameworks. One could conceivably use it to accelerate a UI framework's internals, and you could use it in a very MobX-meets-Redux way by having the return value being serialized state to send to the view, and sending a proxy to the view that lets you invoke various traps. And the `static` variables are your obvious state.
- It glides right in perfectly with https://github.com/tc39/proposal-eventual-send, providing a very nice and easy backend while letting those act as a front end. It might be worth adding a built-in wrapper for these as a follow-on if that takes off, or maybe even having async actor instances implement `[[EventualSend]]` and so on. One could also imagine using it as the basis of something similar to Cloudflare's durable objects.

## Examples

Counter model with corresponding view:

```js
// Model
actor function CounterModel(initial) {
    static let count = static.ignore(initial, initial)

    on increment() {
        count++
    }

    on decrement() {
        count--
    }

    return count
}

// View
function Counter({initial}) {
    const model = useMemo(() => CounterModel(initial), [])
    const [count, setCount] = useState(() => model.render())

    useEffect(() => {
        const subscription = model.subscribe(() => setCount(model.render()))
        return () => subscription.unsubscribe()
    }, [])

    return (
        <div>
            <button onClick={() => model.send.decrement()}>-</button>
            <span>{count}</span>
            <button onClick={() => model.send.increment()}>+</button>
        </div>
    )
}
```

Hypothetical React API update:

```js
// Component
actor function Component(props) {
    // ...
}

// useState
static let state = init

// useMemo, useCallback
static const state = init

// useRef
static const ref = React.createRef()

// useEffect - `signal` is an `AbortSignal`
on effect(signal) {
    // ...
}

// useLayoutEffect
on layoutEffect() {
    // ...
}

// useContext
const value = this.context(MyContext)

// useImperativeHandle
this.handle(ref, () => handle, deps={})

// useReducer becomes largely useless.
actor function model() {
    static let count = 0
    on increment() { count++ }
    on decrement() { count-- }
    return count
}

actor function Counter() {
    static const counter = model()
    static { counter.subscribe(() => { static.update() }) }
    return (
        <>
            Count: {counter.render()}
            <button onClick={() => counter.send.decrement()}>-</button>
            <button onClick={() => counter.send.increment()}>+</button>
        </>
    );
}
```

## Follow-on: Control flow

> This is a follow-on because it's considerably more complex

Non-static contexts cannot use actor-specific productions except at the top level. Static contexts have a different set of rules, which enable it great power.

```js
// When the statement executes (during rendering if top level, when the block
// runs if in a `static` block), if the branch changes, `use bar()` gets
// created, then `use foo()` gets closed. Likewise, if `foo` changes value and
// the same branch is taken, the new actor gets created then the old deleted.
if (cond) {
    use foo()
} else {
    use bar()
}

// When the statement executes (during rendering if top level, when the block
// runs if in a `static` block), if the number of iterations change, new actors
// are initialized, then previous actors are removed.
for (const value of coll) {
    use foo()
}
for (let i = 0; i < limit; i++) {
    use foo()
}
do {
    use foo()
} while (cond)

// When the statement executes (during rendering if top level, when the block
// runs if in a `static` block), new keys are added then old keys are removed,
// and existing keys are simply retained.
for (const value of coll by key) {
    // ...
}

// When the statement executes (during rendering if top level, when the block
// runs if in a `static` block), if the key changes, the replacement actor is
// created, then the old actor is removed.
for key use foo()
```

### Why?

- There's very little overhead in state updates when it's only a series of simple conditionals, and engines could reuse a lot of code with it.
- Engines could find ways to accelerate keyed diffing and patching, among other things, using native instructions and such, so it can be far faster than a JS-land implementation. It could also even JIT it and make it faster than JS could ever be.
