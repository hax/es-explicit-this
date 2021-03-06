# ECMAScript Proposal: Syntax for Explicitly Naming `this`

> `this` isn't really a keyword, it is the natural "main" function argument in JavaScript.
> - (@spion)[https://github.com/mindeavor/es-pipeline-operator/issues/2#issuecomment-162348536]

This proposal extends the `function` declaration syntax to allow for explicit naming of what is normally called `this`.

```js
Object.defineProperty(User.prototype, {
  get: function fullName() { // original version
    return `${this.firstName} ${this.lastName}`
  },
  configurable: true,
})

// versions use the feature of this proposal

function fullName(this) { // explicit this parameter
  return `${this.firstName} ${this.lastName}`
}

function fullName(this user) { // explicit naming `this` to `user`
  return `${user.firstName} ${user.lastName}`
}

function fullName(this {firstName, lastName}) { // destructuring
  return `${firstName} ${lastName}`
}
```

Its primary use case is to play nicely with the [function bind proposal](https://github.com/zenparsing/es-function-bind), making such functions more readable. For example:

```js
function zip (this array, otherArray) {
  return array.map( (a, i) => [a, otherArray[i]] )
}

function flatten (this subject) {
  return subject.reduce( (a,b) => a.concat(b) )
}

[10,20]
  ::zip( [1,2] )
  ::flatten()
//=> [10, 1, 20, 2]
```

## Motivation

The "keyword" `this` has been one of the greatest sources of confusion for programmers learning and using JavaScript. Fundamentally, it boils down to **two primary reasons**:

1. `this` is an **implicit** parameter, and
2. `this` **implicitly performs variable shadowing** in your functions.

Variable shadowing is true source of why `this` is so confusing to learn. In fact, variable shadowing is a bad practice in general. For example:

```js
function process (obj, name) {
  obj.taskName = name;
  doAsync(function (obj, amount) {
    obj.x += amount;
  });
};
```

In the above code, `obj.x` is referring to a different `obj` than `obj.name`. An experienced programmer would likely think this code is silly. Why name the inner parameter `obj` when there is already another variable in the same scope with the same name?

Yet, this behavior is exactly what `this` proceeds to do. If we were to translate the above example to use method-style functions:

```js
function process (name) {
  this.taskName = name;
  doAsync(function (amount) {
    this.x += amount;
  });
};
```

...we would end up with code *just as bad* as the original example. The second `this` is referring to a different object than the first `this`, even though they have the same name.

However, if we could **explicitly name** the object within the function parameters, we could disambiguate the two to avoid such variable shadowing, and make the above example look something more like this:

```js
function process (this obj, name) {
  obj.taskName = name;
  doAsync(function callback (this result, amount) {
    result.amount += 2;
  });
};
```

Explicit `this` parameter also allow type annotation or parameter decorators be added just like normal parameter.

```ts
// Type annotation (TypeScript, already possible today)
Number.prototype.toHexString = function (this: number) {
  return this.toString(16)
}
```

```ts
// Parameter decorators (future proposal)
Number.prototype.toHexString = function (@toNumber this num) {
  return num.toString(16) // same as Number(num).toString(16)
}
```

## Protect programmers by throwing as early as possible

If a function explicitly names `this`, attempting to use `this` inside the body will throw an error:

```js
function method (this elem, e) {
  console.log("Elem:", elem) // OK
  console.log("this:", this) // <-- Syntax error!
}
```

This behavior fits well with arrow functions, since they don't contain their own `this` in the first place.

```js
function callback (this elem, e) {
  alert(`You entered: ${elem.value}`);
  setTimeout( () => elem.value = '', 1000 ) // OK
  setTimeout( () => this.value = '', 1000 ) // <-- Syntax error!
}
```

As normal, an inner `function` get its own `this`, which you can still choose whether or not to rename:

```js
runTask(function cb (this elem, e) {

  elem.runAsyncTask(function () {
    console.log("Outer elem:", elem) // OK
    console.log("Inner this:", this) // OK
  });
})
```

Class constructor can't use explicit `this` syntax because class constructor can only be used via `new` and `this` in the constructor is never a parameter or argument passed by caller, the `this` value in constructor is generated by the constructor or its base class.

```js
class C {
  constructor(this) { // <-- Syntax error!
    // ...
  }
}
```

If a function has explicitly named its `this` parameter, it could be useful to throw an error when that function gets called without one or used via `new`. For example:

```js
function zip (this array, otherArray) {
  return array.map( (a, i) => [a, otherArray[i]] )
}

zip() //=> Throws a TypeError on invocation,
      //   before `array.map(...)` is run.

new zip() //=> Also throws a TypeError
```

## Alternate Syntax

Matching the [function bind proposal](https://github.com/zenparsing/es-function-bind):

```js
function array::zip (otherArray) {
  return array.map( (a, i) => [a, otherArray[i]] )
}

function subject::flatten () {
  return subject.reduce( (a,b) => a.concat(b) )
}

[10,20]
  ::zip( [1,2] )
  ::flatten()
//=> [10, 1, 20, 2]
```

## Prior Art

- TypeScript [`this` parameter](https://www.typescriptlang.org/docs/handbook/functions.html#this-parameters)
- Java [receiver parameter](https://stackoverflow.com/questions/24291091/why-can-we-use-this-as-an-instance-method-parameter)
- C# [`this` modifier of the first parameter of an extension method](https://stackoverflow.com/questions/4700016/this-parameter-modifier-in-c)
- Python/Perl: need to be specified explicitly as the first parameter of an instance method, could be named freely, though conventionally `self`
