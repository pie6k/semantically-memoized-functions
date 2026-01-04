# ECMAScript Proposal: Function Semantic Memoization

## Status

**Stage:** 0

**Authors:** [Your Name]

**Champions:** [TBD]

**Spec Text:** [TBD]

---

## The problem

Consider the following code:

```js
// Let's have a function that returns another function and uses a variable from parent scope.
function getHandlerOfData(data) {
  function handler(prefix) {
    console.log(prefix, data);
  }
  return handler;
}

// We have some reference to some object
const data = { id: 1 };
// We call the function with the same object twice
const handler1 = getHandlerOfData(data);
const handler2 = getHandlerOfData(data);

// Those two functions are semantically identical, but they are different function references.
```

In the example above, `handler1` and `handler2` are semantically identical. They will absolutely behave identically, there is no observable difference between them besides the fact that they are like copies of each other.

Often times, we want an easy way to re-use the same function reference when we know it is semantically identical to some function created previously.

## Introduction

This proposal introduces syntax for defining **semantically memoized functions** — functions definition that preserve referential identity, if all the variables from parent scope used inside the function are the same.


```js
// The same example as above, but with the new syntax:
function getHandlerOfData(data) {
  // New syntax: function~
  function~ handler(prefix) {
    console.log(prefix, data);
  }
  return handler;
}

// We have some reference to some object
const data = { id: 1 };
// We call the function with the same object twice
const handler1 = getHandlerOfData(data);
const handler2 = getHandlerOfData(data);

handler1 === handler2; // true — same closed-over value, same function reference
```

## Example use cases

### The React Problem

Modern UI frameworks, particularly React, use referential equality extensively for performance optimization:

```jsx
function TodoItem({ todo, onDelete }) {
  // If this component is rendered multiple times, with the same `todo` and `onDelete` props - there is no need to re-create the `handleClick` function every time with a new reference.
  const handleClick = () => {
    onDelete(todo.id);
  };

  // Every render creates a new `handleClick` function
  // This breaks React.memo optimization and causes unnecessary re-renders
  return <Button onClick={handleClick}>Delete</Button>;
}
```

With the new syntax, we can define the `handleClick` function as semantically memoized:

```jsx
function TodoItem({ todo, onDelete }) {
  // The only difference between this and the previous example is the `~>` syntax instead of regular `=>` arrow function syntax.
  const handleClick = () ~> {
    onDelete(todo.id);
  };

  // Now, for every render with the same `todo` and `onDelete` props - the same `handleClick` function will be used.
  // Button will not re-render unnecessarily.
  return <Button onClick={handleClick}>Delete</Button>;
}
```

Currently, the solution is to use the `useCallback` hook or React compiler that might work, but is not a universal JavaScript solution.

```jsx
function TodoItem({ todo, onDelete }) {
  const handleClick = useCallback(() => {
    onDelete(todo.id);
  }, [todo.id, onDelete]); // Manual dependency array

  return <Button onClick={handleClick}>Delete</Button>;
}
```

This approach has significant drawbacks:

1. **Verbose boilerplate** — Every memoized function requires wrapping in `useCallback`
2. **Manual dependency tracking** — Developers must manually specify dependencies, which is error-prone
3. **Stale closure bugs** — Forgetting a dependency leads to subtle bugs where functions capture outdated values
4. **Linter dependency** — The `eslint-plugin-react-hooks` exists primarily to catch dependency array mistakes
5. **Framework-specific** — This is a React-specific solution to a general JavaScript problem


---

## Transpilation to JavaScript

It might be easy to understand the underlying idea by seeing how it would be transpiled to current JavaScript:

This code:

```js
function getHandlerOfData(data) {
  // New syntax: function~
  function~ handler(prefix) {
    console.log(prefix, data);
  }
  return handler;
}
```

Could be transpiled to something like this:

```js
// Some top level cache created for each semantically memoized function
const __cache_getHandlerOfData = new __FunctionMemoizationCache();

function getHandlerOfData(data) {
  // Transpiler knows what variables from the parent scope are used inside the function and will use them as cache keys.
  const __handler__keys = [data];

  // Before creating the function, we check if there is a cached version of it
  let handler = __cache_getHandlerOfData.get(__handler__keys);
  
  // If there is no cached version, we create a new one and store it in the cache for future use.
  if (!handler) {
    handler = function handler(prefix) {
      console.log(prefix, data);
    };
    __cache_getHandlerOfData.set(__handler__keys, handler);
  }

  return handler;
}
```

Summary of what happens:
- Transpiler knows what variables from the parent scope are used inside the function and will use them as cache keys.
- Before creating the function, we check if there is a cached version of it
- If there is no cached version, we create a new one and store it in the cache for future use.
- If there is a cached version, we return the cached function reference.

### Syntax

#### Function Declarations

```js
function~ memoized(arg) {
  return closedOverValue + arg;
}
```

#### Arrow Functions

```js
const memoized = (arg) ~> closedOverValue + arg;
```

#### Function Expressions

```js
const memoized = function~(arg) {
  return closedOverValue + arg;
};

const named = function~ namedMemoized(arg) {
  return closedOverValue + arg;
};
```

### Cache key only uses variables that are actually used inside the child function

```js
const someVariable = {};

function parent(a, b, c) {
  function~ inner() {
    console.log(a); // We have multiple variables in scope, but only `a` is used inside the function, so the cache key is only [a].
  }
  return inner;
}

// Cache key is [a], not [a, b, c]
parent(1, 'x', {}) === parent(1, 'y', []); // true — same `a` value
```

### Equality Semantics

Cache keys can be both primitive and object values.

For cache key comparison:

- **Primitives** — Compared by value (`===`)
- **Objects** — Compared by reference (`===`)

```js
function parent(obj, num) {
  function~ inner() {
    console.log(obj, num);
  }
  return inner;
}

const myObj = { id: 1 };

parent(myObj, 42) === parent(myObj, 42);     // true
parent({ id: 1 }, 42) === parent({ id: 1 }, 42); // false — different object references
parent(myObj, 42) === parent(myObj, 43);     // false — different primitive value
```


## Detailed Semantics

### Variable Analysis

The set of cache key variables is determined by static analysis of the function body. Only variables that are:

1. Defined in an enclosing scope (not the function's own scope)
2. Actually referenced in the function body

are included in the cache key.

```js
function outer(a, b, c) {
  const local = 'ignored';

  function~ inner(param) {
    const innerLocal = a + param; // `a` is from outer scope
    return innerLocal + b;        // `b` is from outer scope
    // `c` is never used
    // `local` is never used
    // `param` is inner's own parameter
    // `innerLocal` is inner's own variable
  }

  return inner;
}

// Cache key: [a, b]
```

### How does this interact with closures over mutable values?

In the example below, `count` is a mutable value and on each call, `increment` has access to 'fresh' value of `count`.



```js
let count = 0;

function outer() {

  function~ increment() {
    // Syntax error - `count` cannot be used as it is a mutable value
    return ++count;
  }

  return increment;
}
```

This is a valid code and would be transpiled to something like this:

```js
const __cache_outer = new __FunctionMemoizationCache();

let count = 0;
const __count__ref = { get current() { return count; }, set current(value) { count = value; } };

function outer() {
  const __increment__keys = [__count__ref];

  let increment = __cache_outer.get(__increment__keys);

  if (!increment) {
    increment = function~ increment() {
      return ++count;
    };
    __cache_outer.set(__increment__keys, increment);
  }

  return increment;
}
```

### Nested Memoized Functions

Memoized functions can be nested. Each level maintains its own cache:

```js
function grandparent(x) {
  function~ parent(y) {
    function~ child(z) {
      return x + y + z;
    }
    return child;
  }
  return parent;
}

const p1 = grandparent(1);
const p2 = grandparent(1);
p1 === p2; // true — same `x`

const c1 = p1(2);
const c2 = p1(2);
c1 === c2; // true — same `x` (via parent), same `y`
```

### `this` Binding

For non-arrow memoized functions, `this` is **not** part of the cache key. The function identity is determined solely by closed-over variables:

```js
function createMethod(multiplier) {
  return function~(value) {
    return this.base * multiplier * value;
  };
}

const method = createMethod(2);
const obj1 = { base: 10, calc: method };
const obj2 = { base: 20, calc: method };

obj1.calc === obj2.calc; // true — same function reference
obj1.calc(5); // 100 (10 * 2 * 5)
obj2.calc(5); // 200 (20 * 2 * 5)
```

For arrow functions, `this` is captured from the enclosing scope and **is** part of the cache key if used:

```js
function createMethod(multiplier) {
  return (value) ~> {
    return this.base * multiplier * value; // `this` is closed over
  };
}

// Cache key includes `this` (from parent scope, as arrow functions will not override `this` value) and `multiplier`
```

### Arguments and Rest Parameters

The function's own parameters, `arguments` object, and rest parameters are never part of the cache key — they belong to the function's own scope:

```js
function outer(config) {
  function~ inner(...args) {
    return config.process(args);
  }
  return inner;
}

// Cache key: [config]
// `args` is not included — it's the function's own rest parameter
```

---

## Implementation Strategy

Creating transpilation strategy is not trivial, but it is possible.


### Memory management

Cache internal structure is a mixture of `WeakMap` and `Map`. If some variables are primitives, cache would be stored forever. If some variables are objects, cache would be stored with weak references. If those objects are no longer reachable and garbage collected, the cache entry will be removed.




## Async and Generator Variants

For completeness, the syntax extends to async functions and generators:

### Async Functions

```js
async function~ fetchData(url) {
  const response = await fetch(url);
  return response.json();
}

const asyncArrow = async (url) ~> {
  const response = await fetch(url);
  return response.json();
};
```

### Generator Functions

```js
function~* generateRange(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}
```

### Async Generators

```js
async function~* streamData(source) {
  for await (const chunk of source) {
    yield processChunk(chunk);
  }
}
```

The semantics remain the same: function identity is memoized based on closed-over variables.

---

## What is not included in this proposal

## Class Methods

Class method syntax is **not** included in this proposal. Class methods are already defined once on the prototype and shared across all instances:

```js
class Counter {
  count = 0;

  increment() {
    this.count++;
  }
}

const a = new Counter();
const b = new Counter();
a.increment === b.increment; // true — same method on prototype
```

The function identity issue this proposal addresses occurs specifically with functions created inside other function calls, not with prototype methods.

---

## Object Method Shorthand

Object literal method shorthand is **not** included in this initial proposal:

```js
// NOT proposed (may be considered for future extension)
const obj = {
  method~(x) { return x + this.value; }
};
```

This could be explored as a follow-on proposal if there is community interest.

---

## Design Decisions

### Why New Syntax Instead of a Decorator or Function?

**Syntactic approach benefits:**

1. **Static analysis** — Engines can determine closed-over variables at parse time
2. **No runtime overhead** — No wrapper function calls
3. **Clear semantics** — The `~` clearly marks memoized function boundaries
4. **Optimizer-friendly** — Engines can apply optimizations knowing the semantic intent

A function-based approach (`memoizeIdentity(() => {...})`) cannot statically determine closed-over variables and requires runtime introspection or manual dependency specification.

### Why `~` Symbol?

- Visually lightweight, doesn't obscure the function signature
- Suggests "approximately" or "equivalent to" — fitting for semantic equivalence
- Available and unambiguous in current grammar
- Similar position to `*` in generators (`function*`), establishing precedent

### Why Not Memoize Return Values?

This proposal specifically memoizes **function identity**, not return values. Return value memoization is a different optimization with different trade-offs that is out of scope of this proposal.

### Purity Assumptions

This feature is most useful with pure functions, but does not enforce purity. A memoized function that performs side effects based on mutable closed-over state is valid JavaScript but may produce unexpected results:

```js
let counter = 0;

function outer() {
  function~ inner() {
    return ++counter; // Impure — mutates external state
  }
  return inner;
}

const fn = outer();
fn(); // 1
fn(); // 2
outer()(); // 3 — same function reference, continues counting
```

This mirrors existing JavaScript patterns — developers are responsible for understanding the implications of their code.

---

## FAQ

### What about recursive memoized functions?

Works as expected. The function reference is established before the body executes:

```js
function outer(n) {
  function~ factorial(x) {
    return x <= 1 ? 1 : x * factorial(x - 1);
  }
  return factorial(n);
}
```

### Can I mix memoized and non-memoized functions?

Yes. The syntax applies per-function:

```js
function parent(x) {
  function~ memoized() { return x; }
  function regular() { return x; }

  return { memoized, regular };
}

const a = parent(1);
const b = parent(1);
a.memoized === b.memoized; // true
a.regular === b.regular;   // false
```

### What is the memory impact?

Cached functions are held weakly with respect to their object keys. When closed-over objects become unreachable, associated cache entries can be garbage collected. For purely primitive keys, entries persist indefinitely (similar to string interning).

### Does this work with `eval` or `new Function`?

Memoized functions created via `eval` or `new Function` are not statically analyzable and would not benefit from memoization, though exact semantics would be determined during specification.

---

## Prior Art

### React Hooks

React's `useCallback` and `useMemo` provide similar functionality but require manual dependency arrays and are framework-specific.

### Kotlin's Lambda Caching

Kotlin caches lambda expressions that don't capture mutable state, achieving similar referential stability.

### Swift's Closure Optimization

Swift's compiler can optimize closures that capture values by copy, sometimes reusing closure instances.

### Userland Libraries

- **Reselect** — Memoized selectors for Redux
- **memoize-one** — Memoizes based on most recent arguments
- **React Compiler (experimental)** — Automatic memoization through compilation

---

## Related Proposals

- [Records and Tuples](https://github.com/tc39/proposal-record-tuple) — Immutable data structures with value equality
- [Pattern Matching](https://github.com/tc39/proposal-pattern-matching) — Structural matching

---

## Specification

[TBD — Formal specification text will be developed as the proposal advances]

---

## References

- [TC39 Process Document](https://tc39.es/process-document/)
- [React useCallback Documentation](https://react.dev/reference/react/useCallback)
- [React Compiler](https://react.dev/learn/react-compiler)

---

## Changelog

- **v0.1** — Initial proposal draft