# Security use case
Node.js’s runtime depends on many built-in JavaScript global intrinsic objects
that are vulnerable to mutation or prototype pollution by third-party libraries.
When initializing a JavaScript runtime, Node.js therefore caches
wrapped versions of every global intrinsic object (and its methods)
in a [large `primordials` object][primordials.js].

Many of the global intrinsic methods inside of the `primordials` object
rely on the `this` binding.
`primordials` therefore contains numerous entries that look like this:
```js
ArrayPrototypeConcat: uncurryThis(Array.prototype.concat),
ArrayPrototypeCopyWithin: uncurryThis(Array.prototype.copyWithin),
ArrayPrototypeFill: uncurryThis(Array.prototype.fill),
ArrayPrototypeFind: uncurryThis(Array.prototype.find),
ArrayPrototypeFindIndex: uncurryThis(Array.prototype.findIndex),
ArrayPrototypeLastIndexOf: uncurryThis(Array.prototype.lastIndexOf),
ArrayPrototypePop: uncurryThis(Array.prototype.pop),
ArrayPrototypePush: uncurryThis(Array.prototype.push),
ArrayPrototypePushApply: applyBind(Array.prototype.push),
ArrayPrototypeReverse: uncurryThis(Array.prototype.reverse),
```
…and so on, where `uncurryThis` is `Function.prototype.call.bind`
(also called [“call-binding”][call-bind]),
and `applyBind` is the similar `Function.prototype.apply.bind`.

[call-bind]: https://npmjs.com/call-bind

In other words, Node.js must **wrap** every `this`-sensitive global intrinsic method
in a `this`-uncurried **wrapper function**,
whose first argument is the method’s `this` value,
using the `uncurryThis` helper function.

The result is that code that uses these global intrinsic methods,
like this code adapted from [node/lib/internal/v8_prof_processor.js][]:
```js
  // `specifier` is a string.
  const file = specifier.slice(2, -4);

  // Later…
  if (process.platform === 'darwin') {
    tickArguments.push('--mac');
  } else if (process.platform === 'win32') {
    tickArguments.push('--windows');
  }
  tickArguments.push(...process.argv.slice(1));
```
…must instead look like this:
```js
// Note: This module assumes that it runs before any third-party code.
const {
  ArrayPrototypePush,
  ArrayPrototypePushApply,
  ArrayPrototypeSlice,
  StringPrototypeSlice,
} = primordials;

  // Later…
  const file = StringPrototypeSlice(specifier, 2, -4);

  // Later…
  if (process.platform === 'darwin') {
    ArrayPrototypePush(tickArguments, '--mac');
  } else if (process.platform === 'win32') {
    ArrayPrototypePush(tickArguments, '--windows');
  }
  ArrayPrototypePushApply(tickArguments, ArrayPrototypeSlice(process.argv, 1));
```

This code is now protected against prototype pollution by accident and by adversaries
(e.g., `delete Array.prototype.push` or `delete Array.prototype[Symbol.iterator]`).
However, this protection comes at two costs:

1. These [uncurried wrapper functions sometimes dramatically reduce performance][#38248].
   This would not be a problem if Node.js could cache
   and use the intrinsic methods directly.
   But the only current way to use intrinsic methods
   would be with `Function.prototype.call`, which is also vulnerable to mutation.

2. The Node.js community has had [much concern about barriers to contribution][#30697]
   by ordinary JavaScript developers, due to the unidiomatic code encouraged by these
   uncurried wrapper functions.

Both of these problems are much improved by the bind-`this` operator.
Instead of wrapping every global method with `uncurryThis`,
Node.js could cached and used **directly**
without worrying about `Function.prototype.call` mutation:

```js
// Note: This module assumes that it runs before any third-party code.
const $apply = Function.prototype.apply;
const $push = Array.prototype.push;
const $arraySlice = Array.prototype.slice;
const $stringSlice = String.prototype.slice;

  // Later…
  const file = specifier->$stringSlice(2, -4);

  // Later…
  if (process.platform === 'darwin') {
    tickArguments->$push('--mac');
  } else if (process.platform === 'win32') {
    tickArguments->$push('--windows');
  }
  $push->$apply(tickArguments, process.argv->$arraySlice(1));
```

Performance has improved, and readability has improved.
There are no more uncurried wrapper functions;
instead, the code uses the intrinsic methods in a notation
similar to normal method calling with `.`.

[node/lib/internal/v8_prof_processor.js]: https://github.com/nodejs/node/blob/e46c680bf2b211bbd52cf959ca17ee98c7f657f5/lib/internal/v8_prof_processor.js
[#38248]: https://github.com/nodejs/node/pull/38248
[#30697]: https://github.com/nodejs/node/issues/30697
