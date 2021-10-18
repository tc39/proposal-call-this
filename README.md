# Bind-`this` operator for JavaScript
ECMAScript Stage-0 Proposal. J. S. Choi, 2021.

* **[Formal specification][]**
* Babel plugin: Not yet

[formal specification]: http://jschoi.org/21/es-bind-operator/

This proposal is a **resurrection**
of the old [Stage-0 bind-operator proposal][old bind].
It is also an alternative, **competing** proposal
to the [Stage-1 extensions proposal][extensions].
For more information, see [§ Related proposals](#related-proposals).

[old bind]: https://github.com/tc39/proposal-bind-operator
[extensions]: https://github.com/tc39/proposal-extensions

## Syntax
```js
receiver::fn
receiver::ns.fn
receiver::(expr)
```

<dl>

<dt><code>receiver</code></dt>
<dd>

A member expression, a call expression, an optional expression,
a `new` expression with arguments, another bind-`this` expression,
or a parenthesized expression.
The value to which this expression resolves will be bound
to the right-hand side’s function object, as that function’s `this` receiver.

</dd>

<dt><code>fn</code></dt>
<dd>

A variable that must resolve to a function object.

</dd>

<dt><code>ns</code></dt>
<dd>

Instead of a single variable,
the right-hand side may be a namespace object’s variable,
followed by a chain of property identifiers.
This chain must resolve to a function object.

</dd>

<dt><code>expr</code></dt>
<dd>

An arbitrary expression within parentheses,
which must resolve to a function object.

</dd>

</dl>

## Description
(A [formal specification][] is available.)

The bind-`this` operator `::` is a **left-associative** binary operator.
It binds its left-hand side (a **receiver** value)
to its right-hand side (a **function**)’s `this` value,
creating a **bound function** in the same manner
as [`Function.prototype.bind`][bind].

For example, `receiver::fn` would be equivalent to `fn.bind(receiver)`
(except that its behavior does not change
if code elsewhere reassigns the global method `Function.prototype.bind`).

Likewise, `receiver::(createMethod())` would be roughly
equivalent to `createMethod().bind(receiver)`.

If the operator’s right-hand side does not evaluate to a function during runtime,
then the program throws a TypeError.

The bound functions created by the bind-`this` operator
are **indistinguishable** from the bound functions
that are already created by [`Function.prototype.bind`][bind].
Both are **exotic objects** that do not have a `prototype` property,
and which may be called like any typical function.

From this definition, `receiver::fn(...args)`
is also **indistinguishable** from `fn.call(receiver, ...args)`
(except that its behavior does not change
if code elsewhere reassigns the global method `Function.prototype.call`).

The operator’s **left** side has equal **[precedence][]** with
member expressions, call expressions, `new` expressions with arguments,
and optional expressions.
Like those operators, the bind-`this` operator also may be short-circuited
by optional expressions in its left-hand side.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                     | Example       | Grouping
| ---------------------------------- | ------------- | -----------------
| Member expressions                 |`obj.prop::fn` |`(obj.prop)::fn`
| Call expressions                   |`obj()::fn`    |`(obj())::fn`
| Optional expressions               |`obj?.prop::fn`|`(obj?.prop)::fn`
|`new` expressions with arguments    |`new C()::fn`  |`(new C())::fn`

The operator’s **right**-hand side, as with decorators, may be one of the following:
* A single **identifier** (like `fn`).
* A **chain** of identifiers (like `ns.fn`).
* A parenthesized **expression** (like `(createFn())`).

For example, `receiver::ns.ns.ns.fn` groups as `receiver::(ns.ns.ns.fn)`.

Similarly to the `.` and `?.` operators,
the bind-`this` operator may be **padded** by whitespace.\
For example, `receiver :: fn`\
is equivalent to `receiver::fn`,\
and `receiver :: (createFn())`\
is equivalent to `receiver::(createFn())`.

A `new` expression may **not** contain a bind-`this` expression without parentheses.
Both `new x::fn()` and `new x::fn` are SyntaxErrors.
Otherwise, `new x::fn()` would be visually ambiguous between\
`(new x)::fn()`, `new (x::fn)()`, and `new (x::fn())`.

An optional expression may **not** contain a bind-`this` expression without parentheses.
For example, `receiver::x?.prop`, `receiver::x?.[key]`, and `receiver::x?.()`
are all SyntaxErrors.
Otherwise, `receiver::x?.prop` would be visually ambiguous
between `(receiver::x)?.prop` and `receiver::(x?.prop)`.
(And both of these possibilities would be useless anyway:\
A bind-`this` expression will **never** evaluate to `null` or `undefined`,\
so `(receiver::x)?.prop` is not useful.\
And `receiver::(null)` and `receiver::(undefined)` will always throw TypeErrors,\
so `receiver::(x?.prop)` is also not useful.)

## Why a bind-`this` operator
In short:

1. [`.bind`][bind], [`.call`][call], and [`.apply`][apply]
   are very useful and very common in JavaScript codebases.
2. But `.bind`, `.call`, and `.apply` are clunky and unergonomic.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call
[apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply

### `.bind`, `.call`, and `.apply` are very common
The dynamic `this` binding is a fundamental part of JavaScript design and practice today.
Because of this, developers frequently need to change the `this` binding.
`.bind`, `.call`, and `.apply` are arguably three of the most commonly used functions
in all JavaScript.

We can estimate `.bind`, `.call`, and `.apply`’s prevalences using [Node Gzemnid][].
Although [Gzemnid can be deceptive][], we are only seeking rough estimations.

[Node Gzemnid]: https://github.com/nodejs/Gzemnid
[Gzemnid can be deceptive]: https://github.com/nodejs/Gzemnid/blob/main/README.md#deception

First, we download the 2019-06-04 [pre-built Gzemnid dataset][]
for the top-1000 downloaded NPM packages.
We also need Gzemnid’s `search.topcode.sh` script in the same active directory,
which in turn requires the lz4 command suite.
`search.topcode.sh` will output lines of code from the top-1000 packages
that match the given regular expression.

[pre-built Gzemnid dataset]: https://gzemnid.nodejs.org/datasets/

```bash
./search.topcode.sh '\.call\b' | head
grep -aE  "\.call\b"
177726827	debug@4.1.1/src/common.js:101:					match = formatter.call(self, val);
177726827	debug@4.1.1/src/common.js:111:			createDebug.formatArgs.call(self, args);
154772106	kind-of@6.0.2/index.js:54:  type = toString.call(val);
139612972	readable-stream@3.4.0/errors-browser.js:26:      return _Base.call(this, getMessage(arg1, arg2, arg3)) || this;
139612972	readable-stream@3.4.0/lib/_stream_duplex.js:60:  Readable.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_duplex.js:61:  Writable.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_passthrough.js:34:  Transform.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_readable.js:183:  Stream.call(this);
139612972	readable-stream@3.4.0/lib/_stream_readable.js:786:  var res = Stream.prototype.on.call(this, ev, fn);
```

We use `awk` to count those matching lines of code
and compare their numbers for `bind`, `call`, `apply`,
and several other frequently used functions.

```bash
> ls
search.topcode.sh
slim.topcode.1000.txt.lz4
> ./search.topcode.sh '\.call\b' | awk 'END { print NR }'
500084
> ./search.topcode.sh '\.apply\b' | awk 'END { print NR }'
225315
> ./search.topcode.sh '\.bind\b' | awk 'END { print NR }'
170248
> ./search.topcode.sh '\b.map\b' | awk 'END { print NR }'
1016503
> ./search.topcode.sh '\bconsole.log\b' | awk 'END { print NR }'
271915
> ./search.topcode.sh '\.slice\b' | awk 'END { print NR }'
182292
> ./search.topcode.sh '\.set\b' | awk 'END { print NR }'
168872
> ./search.topcode.sh '\.push\b' | awk 'END { print NR }'
70116
```

These results suggest that usage of `.call`, `.bind`, and `.apply`
are comparable to usage of other frequently used standard functions.
In this dataset, their combined usage even exceeds that of `console.log`.

Obviously, [this methodology has many pitfalls][Gzemnid can be deceptive],
but we are only looking for roughly estimated orders of magnitude
relative to other baseline functions.
Gzemnid counts each library’s codebase only once; it does not double-count dependencies.

In fact, this method definitely underestimates the prevalences
of `.bind`, `.call`, and `.apply`
by excluding the large JavaScript codebases of Node and Deno.
Node and Deno [copiously use bound functions for security][security-use-case]
hundreds or thousands of times.

[security-use-case]: https://github.com/js-choi/proposal-bind-this/blob/main/security-use-case.md

### `.bind`, `.call`, and `.apply` are clunky
JavaScript developers are used to using methods in a [noun–verb–noun word order][]
that resembles English and other [SVO human languages][]: `receiver.method(arg)`.

[SVO human languages]: https://en.wikipedia.org/wiki/Category:Subject–verb–object_languages

However, `.bind`, `.call`, and `.apply` flip this “natural” word order,
They flip the first noun and the verb,
and they interpose the verb’s `Function.prototype` method between them:
`method.call(receiver, arg)`.

Consider the following real-life code using `.bind` or `.call`,
and compare them to versions that use the bind-`this` operator.
The difference is especially evident when you read them aloud.

```js
// kind-of@6.0.2/index.js
type = toString.call(val);
type = val::toString();

// debug@4.1.1/src/common.js
match = formatter.call(self, val);
match = self::formatter(val);

createDebug.formatArgs.call(self, args);
self::createDebug.formatArgs(args);

// pretty-format@24.8.0/build-es5/index.js
var code = fn.apply(colorConvert, arguments);
var code = colorConvert::fn(...arguments);

// q@1.5.1/q.js
return value.apply(thisp, args);
return thisp::value(...args);

// rxjs@6.5.2/src/internal/operators/every.ts
result = this.predicate.call(this.thisArg, value, this.index++, this.source);
result = this.thisArg::this.predicate(value, this.index++, this.source);

// bluebird@3.5.5/js/release/synchronous_inspection.js
return isPending.call(this._target());
return this._target()::isPending();

var matchesPredicate = tryCatch(item).call(boundTo, e);
var matchesPredicate = boundTo::(tryCatch(item))(e);

// graceful-fs@4.1.15/polyfills.js
return fs$read.call(fs, fd, buffer, offset, length, position, callback)
return fs::fs$read(fd, buffer, offset, length, position, callback)
```

[noun–verb–noun word order]: https://en.wikipedia.org/wiki/Subject–verb–object

## Non-goals
A goal of this proposal is **simplicity**.
Therefore, this proposal purposefully
does *not* address the following use cases.

**Method extraction with implicit binding**
is not a goal of this proposal.
When extracting a method from an object
and then calling it on the **same** object
requires that that object be **repeated**.
```js
const { fn } = receiver;
receiver::fn(...args);
```
This is **not** a big problem.
In general, methods are extracted from **prototype** objects
and then called on **instance** objects,
so the extraction’s source object and the call’s receiver object
are usually different anyway.
const { slice } = Array.prototype;
arr::slice(...args);

We are deferring any special syntax
for method extraction with implicit binding
to a future proposal.

**Extracting property accessors** (i.e., getters and setters)
is not a goal of this proposal.
Get/set accessors are **not like** methods.
Methods are **properties** (which happen to be functions).
Accessors themselves are **not** properties;
they are functions that activate when getting or setting properties.
Getters/setters have to be extracted using `Object.getOwnPropertyDescriptor`;
they are not handled in a special way.
This verbosity may be considered to be desirable [syntactic salt][]:
it makes the developer’s intention (to extract getters/setters – and not methods)
more explicit.

```js
const { get: $getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// The adversary’s code.
delete Set; delete Function;

// Our own trusted code, running later.
new Set([0, 1, 2])::$getSize();
```

[syntactic salt]: https://en.wikipedia.org/wiki/Syntactic_sugar#Syntactic_salt

**Function/expression application**,
in which deeply nested function calls and other expressions
are untangled into linear pipelines,
is important but not addressed by this proposal.
Instead, it is addressed by the [pipe operator][],
with which this proposal’s syntax works well.

[pipe operator]: #pipe-operator

## Related proposals

### Old bind operator
This proposal is a **resurrection**
of the old [Stage-0 bind-operator proposal][old bind].
A champion of the old proposal has [recommended restarting with a new proposal][fresh]
instead of using the old proposal.

[fresh]: https://github.com/tc39/proposal-bind-operator/issues/56#issuecomment-698444297

This means it is basically the same as the old proposal.
The only big difference is that there is no unary form
for implciit binding of the receiver during method extraction.
(See also [non-goals](#non-goals).)

### Extensions
The extensions system is an alternative, **competing** proposal
to the [Stage-1 extensions proposal][extensions].

An [in-depth comparison is also available][extensions compare].
The concrete differences briefly are:

1. Bind-`this` has no special variable namespace.
2. Bind-`this` has no implicit syntactic handling of property accessors.
3. Bind-`this` has no polymorphic `const ::{ … } from …;` syntax.
4. Bind-`this` has no polymorphic `…::…:…` syntax.
5. Bind-`this` has no `Symbol.extension` metaprogramming system.

[extensions compare]: https://github.com/js-choi/proposal-bind-this/blob/main/extensions-comparison.md

### Pipe operator
The [pipe operator][pipe repo] is a **complementary** proposal
that can be used to linearize deeply nested expressions
like `f(0, g([h()], 1), 2)` into `h() |> g(^, 1) |> f(0, ^, 2)`.

This is fundamentally different than the bind-`this` operator’s purpose,
which would be much closer to property access `.`.

It is true that property access `.`, bind-`this`, and the pipe operator
all may be used to linearize code.
But this is a mere happy side effect for the first two operators:

* Property access is tightly coupled to object membership.
* Bind-`this` is simply changes the `this` binding of a function.

In contrast, the pipe operator is designed to generally
linearize all other kinds of expressions.

Just like how the pipe operator coexists with property access:
```js
// Adapted from react@17.0.2/scripts/jest/jest-cli.js
Object.keys(envars)
  .map(envar => `${envar}=${envars[envar]}`)
  .join(' ')
  |> `$ ${^}`
  |> chalk.dim(^, 'node', args.join(' '))
  |> console.log(^);
```

…so too can it coexist with bind-`this`:
```js
// Adapted from chalk@2.4.2/index.js
return this._styles
  |> (^ ? ^.concat(codes) : [codes])
  |> this::build(^, this._empty, key);
```

[pipe repo]: https://github.com/tc39/proposal-pipeline-operator
