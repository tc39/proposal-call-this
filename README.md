# Call-this operator for JavaScript
ECMAScript Stage-1 Proposal. J. S. Choi, 2021.

* **[Formal specification][]**
* Babel plugin: Not yet

[formal specification]: http://jschoi.org/21/es-bind-operator/

This proposal is a **resurrection** of the old [Stage-0 bind-operator
proposal][old bind]. It is also an alternative, **competing** proposal to the
[Stage-1 extensions proposal][extensions]. For more information, see [§ Related
proposals](#related-proposals).

[old bind]: https://github.com/tc39/proposal-bind-operator
[extensions]: https://github.com/tc39/proposal-extensions

## Syntax

The syntax is being bikeshedded in [issue #10][].

[issue #10]: https://github.com/tc39/proposal-call-this/issues/10

<details>
<summary>Tentative syntax</summary>

```js
receiver :> fn(arg0, arg1)
receiver :> ns.fn(arg0, arg1)
receiver :> (expr)(arg0, arg1)
```

<dl>

<dt><code>receiver</code></dt>
<dd>

A member expression, a call expression, an optional expression, a `new`
expression with arguments, another call-this expression, or a parenthesized
expression. The value to which this expression resolves will be bound to the
right-hand side’s function object, as that function’s `this` receiver.

</dd>

<dt><code>fn</code></dt>
<dd>

A variable that must resolve to a function object.

</dd>

<dt><code>ns</code></dt>
<dd>

Instead of a single variable, the right-hand side may be a namespace object’s
variable, followed by a chain of property identifiers. This chain must resolve
to a function object.

</dd>

<dt><code>expr</code></dt>
<dd>

An arbitrary expression within parentheses,
which must resolve to a function object.

</dd>

<dt><code>arg0, arg1</code>, etc.</dt>
<dd>

A series of argument expressions, which may include spread `...` syntax.

</dd>

</dl>

</details>

## Description

The syntax is being bikeshedded in [issue #10][].

[issue #10]: https://github.com/tc39/proposal-call-this/issues/10

<details>
<summary>Tentative description</summary>

(A [formal specification][] is available.)

The call-this operator `:>` is a **left-associative** binary operator. It calls
its right-hand side (a **function**), binding its `this` value to its left-hand
side (a **receiver** value), as well well as any given arguments – in the same
manner as [`Function.prototype.call`][call].

For example, `receiver :> fn(arg0, arg1)` would be equivalent to
`fn.call(receiver, arg0, arg1)` (except that its behavior does not change if
code elsewhere reassigns the global method `Function.prototype.call`).

Likewise, `receiver :> (createFn())(arg0, arg1)` would be roughly equivalent to
`createFn().call(receiver)`.

If the operator’s right-hand side does not evaluate to a function during
runtime, then the program throws a TypeError.

The operator’s **left** side has equal **[precedence][]** with member
expressions, call expressions, `new` expressions with arguments, and optional
expressions. Like those operators, the call-this operator also may be
short-circuited by optional expressions in its left-hand side.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                  | Example            | Grouping
| ------------------------------- | ------------------ | --------------------
| Member expressions              |`obj.prop :> fn(a)` |`(obj.prop) :> fn(a)`
| Call expressions                |`obj() :> fn(a)`    |`(obj()) :> fn(a)`
| Optional expressions            |`obj?.prop :> fn(a)`|`(obj?.prop) :> fn(a)`
|`new` expressions with arguments |`new C() :> fn(a)`  |`(new C()) :> fn(a)`

The operator’s **right**-hand side, as with decorators, may be one of
the following:
* A single **identifier** or **private field** (like `fn` or `#field`).
* A **chain** of identifiers and/or private fields (like `ns.fn` or `this.ns.#field`).
* A parenthesized **expression** (like `(createFn())`).

For example, `receiver :> ns.ns.ns.fn` groups as `receiver :> (ns.ns.ns.fn)`.

Similarly to the `.` and `?.` operators,
the call-this operator may be **padded** by whitespace or not.\
For example, `receiver :> fn`\
is equivalent to `receiver:>fn`,\
and `receiver :> (createFn())`\
is equivalent to `receiver:>(createFn())`.

The call-this operator may be optionally chained with `?.` (i.e., `?.:>`):\
`receiver :> fn` will always result in a bound function,
regardless of whether `receiver` is nullish.\
`receiver ?.:> fn` will result in `null` or `undefined`
if `receiver` is `null` or `undefined`.\
`receiver ?.:> fn(arg)` also short-circuits as usual, before `arg` is
evaluated, if `receiver` is nullish.

A `new` expression may **not** contain a call-this expression without
parentheses. `new x :> fn()` is a SyntaxError.
Otherwise, `new x :> fn()` would be visually ambiguous between\
`(new x) :> fn()` and `new (x :> fn())`.

</details>

## Why a call-this operator
In short:

1. [`.call`][call] is very useful and very common in JavaScript codebases.
2. But `.call` is clunky and unergonomic.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call

### `.call` is very common
The dynamic `this` binding is a fundamental part of JavaScript design and
practice today. Because of this, developers frequently need to change the
`this` binding. `.call` is arguably one of the most commonly used functions in
all of JavaScript.

We can estimate `.call`’s prevalences using [Node Gzemnid][]. Although [Gzemnid
can be deceptive][], we are only seeking rough estimations.

[Node Gzemnid]: https://github.com/nodejs/Gzemnid
[Gzemnid can be deceptive]: https://github.com/nodejs/Gzemnid/blob/main/README.md#deception

The following results are from the checked-in-Git source code of the top-1000
downloaded NPM packages.

| Occurrences | Method      |
| ----------: | ----------- |
| 1,016,503   |`.map`       |
| 315,922     |**`.call`**  |
| 271,915     |`console.log`|
| 182,292     |`.slice`     |
| 170,248     |`.bind`      |
| 168,872     |`.set`       |
| 70,116      |`.push`      |

These results suggest that usage of `.call` is comparable to usage of other
frequently used standard functions. In this dataset, its usage exceeds even
that of `console.log`.

Obviously, [this methodology has many pitfalls][Gzemnid can be deceptive], but
we are only looking for roughly estimated orders of magnitude relative to other
baseline functions. Gzemnid counts each library’s codebase only once; it does
not double-count dependencies.

<details>

<summary>What methodology was used to get these results?</summary>

***

First, we download the 2019-06-04 [pre-built Gzemnid dataset][] for the top-1000
downloaded NPM packages. We also need Gzemnid’s `search.topcode.sh` script in
the same active directory, which in turn requires the lz4 command suite.
`search.topcode.sh` will output lines of code from the top-1000 packages that
match the given regular expression.

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

We use `awk` to count those matching lines of code and compare their numbers
for `call` and several other frequently used functions.

```bash
> ls
search.topcode.sh
slim.topcode.1000.txt.lz4
> ./search.topcode.sh '\.call\b' | grep -E --invert-match '//.*\.call|/\*.+\.call|[^a-zA-Z][A-Z][a-zA-Z0-9_$]*\.call\( *this|_super\.call|_super\.prototype\.|_getPrototypeOf|_possibleConstructorReturn|__super__|WEBPACK VAR INJECTION|_objectWithoutProperties|\.hasOwnProperty\.call' | awk 'END { print NR }'
315922
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

Note that, for `.call`, we use `grep` to exclude several irrelevant occurrences
of `.call` either within comments or from transpiled code. We err on the side of
false exclusions.

| Excluded pattern                             | Reason                                           |
| -------------------------------------------- | ------------------------------------------------ |
| `//.*\.call`                                 | Code comment.                                    |
| `/\*.+\.call`                                | Code comment.                                    |
| `[^a-zA-Z][A-Z][a-zA-Z0-9_$]*\.call\( *this` | Constructor call obsoleted by `super`. See note. |
| `_super\.call`                               | Babel-transpiled `super()` artifact.             |
| `_super\.prototype\.`                        | Babel-transpiled `super.fn()` artifact.          |
| `_objectWithoutProperties`                   | Babel-transpiled `...` artifact.                 |
| `_getPrototypeOf`                            | Babel artifact.                                  |
| `_possibleConstructorReturn`                 | Babel artifact.                                                |
| `__super__`                                  | CoffeeScript artifact.                           |
| `WEBPACK VAR INJECTION`                      | Webpack artifact.                                |
| `\.hasOwnProperty\.call`                     | Obsoleted by `Object.has`.                       |

These excluded patterns were determined by an independent investigator ([Scott
Jamison][]), after [manually reviewing the first 10,000 occurrences of `.call`
in the dataset][review] for why each occurrence occurred.

[review]: https://github.com/tc39/proposal-call-this/issues/12

The excluded `[^a-zA-Z][A-Z][a-zA-Z0-9_$]*\.call\( *this` pattern deserves a
special note. This pattern matches any capitalized identifier followed by
`.call(this`. We exclude any such occurrences because any capitalized identifier
likely refers to a constructor, and using `.call` on a constructor is a pattern
that has largely been obviated by `class` and `super` syntax. It is likely that
this pattern erroneously excludes many legitimate uses of `.call` from our
count, but this bias against `.call` is acceptable for the purposes of rough
comparison.

[Scott Jamison]: https://github.com/theScottyJam

</details>

### `.call` is clunky
JavaScript developers are used to using methods in a [noun–verb–noun word
order][] that resembles English and other [SVO human languages][]. This pattern
is ubiquitous in JavaScript as dot method calls: `receiver.method(arg)`.

[SVO human languages]: https://en.wikipedia.org/wiki/Category:Subject–verb–object_languages

However, `.call` flips this “natural” word order, They flip the first
noun and the verb, and they interpose the verb’s `Function.prototype` method
between them: `method.call(receiver, arg)`.

Consider the following real-life code using `.call`, and compare them
to versions that use the call-this operator. The difference is especially
evident when you read them aloud.

```js
// kind-of@6.0.2/index.js
type = toString.call(val);
type = val :> toString();

// debug@4.1.1/src/common.js
match = formatter.call(self, val);
match = self :> formatter(val);

createDebug.formatArgs.call(self, args);
self :> createDebug.formatArgs(args);

// rxjs@6.5.2/src/internal/operators/every.ts
result = this.predicate.call(this.thisArg, value, this.index++, this.source);
result = this.thisArg :> this.predicate(value, this.index++, this.source);

// bluebird@3.5.5/js/release/synchronous_inspection.js
return isPending.call(this._target());
return this._target() :> isPending();

var matchesPredicate = tryCatch(item).call(boundTo, e);
var matchesPredicate = boundTo :> (tryCatch(item))(e);

// async@3.0.1/internal/initialParams.js
var callback = args.pop(); return fn.call(this, args, callback);
var callback = args.pop(); return this :> fn(args, callback);

// ajv@6.10.0/lib/ajv.js
validate = macro.call(self, schema, parentSchema, it);
validate = self :> macro(schema, parentSchema, it);

// graceful-fs@4.1.15/polyfills.js
return fs$read.call(fs, fd, buffer, offset, length, position, callback)
return fs :> fs$read(fd, buffer, offset, length, position, callback)
```

[noun–verb–noun word order]: https://en.wikipedia.org/wiki/Subject–verb–object

## Non-goals
A goal of this proposal is **simplicity**. Therefore, this proposal
purposefully does *not* address the following use cases:

**Function binding** and **method extraction** are not a goal of this proposal.

Changing the `this` receiver of functions is more common than function binding,
as evidenced by the preceding statistics. Some TC39 representatives have
expressed concern that function binding may be redundant with proposals such as
[PFA (partial function application) syntax][PFA]. Therefore, we will defer
these two features to future proposals.

**Extracting property accessors** (i.e., getters and setters) is also not a
goal of this proposal. Get/set accessors are **not like** methods. Methods are
**properties** (which happen to be functions). Accessors themselves are **not**
properties; they are functions that activate when getting or setting
properties.

Getters/setters have to be extracted using `Object.getOwnPropertyDescriptor`;
they are not handled in a special way. This verbosity may be considered to be
desirable [syntactic salt][]: it makes the developer’s intention (to extract
getters/setters – and not methods) more explicit.

```js
const { get: $getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// The adversary’s code.
delete Set; delete Function;

// Our own trusted code, running later.
new Set([0, 1, 2]) :> $getSize();
```

[syntactic salt]: https://en.wikipedia.org/wiki/Syntactic_sugar#Syntactic_salt

**Function/expression application**, in which deeply nested function calls and
other expressions are untangled into linear pipelines, is important but not
addressed by this proposal. Instead, it is addressed by the [pipe operator][],
with which this proposal’s syntax works well.

[pipe operator]: #pipe-operator

## Related proposals

### Old bind operator
This proposal is a **resurrection** of the old [Stage-0 bind-operator
proposal][old bind]. (A champion of the old proposal has [recommended restarting
with a new proposal][fresh] instead of using the old proposal.)

[fresh]: https://github.com/tc39/proposal-bind-operator/issues/56#issuecomment-698444297

The new proposal is basically the same as the old proposal. The only big
difference is that there is no unary form for implicit binding of the receiver
during method extraction. (See also [non-goals](#non-goals).)

### Extensions
The extensions system is an alternative, **competing** proposal to the [Stage-1
extensions proposal][extensions].

An [in-depth comparison is also available][extensions compare]. The concrete
differences briefly are:

1. Call-this has no special variable namespace.
2. Call-this has no implicit syntactic handling of property accessors.
3. Call-this has no polymorphic `const ::{ … } from …;` syntax.
4. Call-this has no polymorphic `…::…:…` syntax.
5. Call-this has no `Symbol.extension` metaprogramming system.

[extensions compare]: https://github.com/js-choi/proposal-call-this/blob/main/extensions-comparison.md

### Pipe operator
The [pipe operator][pipe repo] is a **complementary** proposal that can be used
to linearize deeply nested expressions like `f(0, g([h()], 1), 2)` into
`h() |> g(^, 1) |> f(0, ^, 2)`.

This is fundamentally different than the call-this operator’s purpose, which
would be much closer to property access `.`.

It is true that property access `.`, call-this, and the pipe operator all may be
used to linearize code. But this is a mere happy side effect for the first two
operators:

* Property access is tightly coupled to object membership.
* Call-this is simply changes the `this` binding of a function call.

In contrast, the pipe operator is designed to generally linearize all other
kinds of expressions.

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

…so too can it work together with call-this:
```js
// Adapted from chalk@2.4.2/index.js
return this._styles
  |> (^ ? ^.concat(codes) : [codes])
  |> this :> build(^, this._empty, key);
```

[pipe repo]: https://github.com/tc39/proposal-pipeline-operator

### PFA syntax
[PFA (partial function application) syntax][PFA] `~()` would tersely create
partially applied functions.

PFA syntax `~()` and call-this `:>` are also complementary and handle different
use cases.

For example, `obj.method~()` would handle method extraction with implicit
binding, which call-this does not address. In other words, when the receiver
object itself contains the function to which we wish to bind, then we need to
repeat the receiver once, with call-this. PFA syntax would allow us to avoid
repeating the receiver.

```js
n.on("click", v.reset.bind(v))
n.on("click", v.reset~())
```

In contrast, call-this changes the receiver of a function call.
`receiver :> fn()`. (This unbound function might have already been extracted
from another object.) PFA syntax does not address this use case.

```js
// bluebird@3.5.5/js/release/synchronous_inspection.js
isPending.call(this._target())
this._target() :> isPending()
```

[PFA]: https://github.com/tc39/proposal-partial-application
