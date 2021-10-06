# The extensions system and the bind-`this` operator
There are two proposals that do similar things.

* A [proposal for an extensions system][extensions system]:
  `::{ extFn } = obj`, `obj::extFn`, and `obj::extNamespace:extFn`.
* A [proposal for a bind-`this` operator][bind-this]: `obj->fn` and `obj->(fn)`.

In brief, the concrete differences are:

1. bind-`this` has no special variable namespace.
2. bind-`this` has no implicit syntactic handling of property accessors.
3. bind-`this` has no polymorphic `const ::{ … } from …;` syntax.
4. bind-`this` has no polymorphic `…::…:…` syntax.
5. bind-`this` has no `Symbol.extension` metaprogramming system.

<table>

<thead>

<th>extensions system
<th>Bind-<code>this</code> operator

</thead>

<tbody>

<tr>
<td>

The extensions system adds
a **special variable namespace**
to every lexical scope,
which is denoted by a `::` sigil.

```js
const ::has = Set.prototype.has;

s::has(1);
```

The first line assigns `Array.prototype.slice`
to a variable `::slice`,
in the special variable namespace.

<td>

In contrast, the bind-`this` operator involves **no** special variable namespace.
The developer needs to make sure
that the variable of the extracted method (`$has`)
does not **[shadow][shadowing]** any other variables.

```js
const $has = Set.prototype.has;

s->$has(1);
```
This is **nothing new**:
developers **already** have to be careful with their variables’ names,
and they have already developed their own naming systems.

<tr>
<td>

The extensions system lets us **extract**
objects’ **property descriptors**
into a **single variable** in the special variable namespace.
We would use **special** syntax
for getting and setting extracted properties.

```js
const ::size =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// Calls the size getter.
s::size;
```
Here, the `::size` variable contains a property descriptor
and is used as if it were a property.

<td>

`Object.getOwnPropertyDescriptor` already lets us
**extract** critical objects’ property getters and setters
into **ordinary** variables.

```js
const { get: getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// Calls the size getter.
s->getSize();
```
We would use the getter/setter functions **as usual**
with no special syntax.

<tr>
<td>

When extracting multiple properties from a prototype,
it is convenient to use destructuring syntax.
The extensions system provides a special **`import`-like**
“destructuring” syntax that allows both
methods and properties with getters/setters
to be treated uniformly.

```js
const ::{
  has,
  add,
  size,
} from Set;
// Automatically extracts from Set.prototype
// because Set is a constructor.

s::has(1);
s::size;
```

<td>

With the bind-`this` operator,
**ordinary** destructuring works just as usual for methods.
In contrast, getters/setters have to be extracted **separately**.
This **verbosity** may be considered to be desirable **[syntactic salt][]**:
it makes the developer’s **intention** (to extract getters/setters – and not methods)
more **explicit**.

```js
const {
  has: $has,
  add: $add,
} = Set.prototype;
const { get: $getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

s->$has(1);
s->$getSize();
```

<tr>
<td>

Furthermore, when [protecting code against prototype pollution][security use case],
this occasional clunkiness may become **moot** anyway
with [Jordan Harband’s **getIntrinsic proposal**][getIntrinsic].

```js
// Our own trusted code,
// running before the adversary.
// We must get these intrinsics separately.
const $has =
  getIntrinsic('Set.prototype.has');
const $add =
  getIntrinsic('Set.prototype.add');
const { get: $getSize } =
  getIntrinsic('Set.prototype.size');
const s = new Set([0, 1, 2]);

// The adversary’s code.
delete Set;
delete Function;

// Our own trusted code, running later.
s::has(1);
s::size;
```

<td>

With getIntrinsic,
extracting property getters (and setters)
becomes as ergonomic as extracting methods.
After all, we have to get the methods separately anyway.

```js
// Our own trusted code,
// running before the adversary.
// We must get these intrinsics separately.
const $has =
  getIntrinsic('Set.prototype.has');
const $add =
  getIntrinsic('Set.prototype.add');
const { get: $getSize } =
  getIntrinsic('Set.prototype.size');
const s = new Set([0, 1, 2]);

// The adversary’s code.
delete Set;
delete Function;

// Our own trusted code, running later.
s->$has(1);
s->$getSize();
```

<tr>
<td>

The extensions system’s `import`-like
“destructuring” syntax also is **polymorphic**.
It changes depending on whether its left-hand side
is a **constructor** or a non-constructor (i.e., **“namespace”**) object.

If the left-hand side is **not** a constructor,
then it calls its right-hand side as a **static method**.

```js
const ::{
  hasOwnProperty as owns,
} from Object;

for (const key in obj) {
  if (o::owns(key)) {
    console.log(key);
  }
}
```

<td>

This static-method-calling functionality overlaps with the **[pipe operator][]**,
which similarly allows postfix chaining.
However, the pipe operator is more versatile:
it is allowed with any kind of expression.

```js
// Our own trusted code,
// running before the adversary.
const {
  hasOwnProperty: owns,
} = Object;

for (const key in obj) {
  if (o |> owns(^, key)) {
    console.log(key);
  }
}
```

<tr>
<td>

The extensions system isolates extracted properties
in their own special variable namespace, with its own lexical name resolution.
The separate lexical namespace has two intended benefits:

* To prevent **confusion** between extracted properties’ names
  and the names of other identifiers.
* To prevent **accidental** extraction of properties
  from a **constructor** object, instead of the constructor’s **prototype**.

```js
const ::{
  has,
  add,
  size,
} from Set;
const s = new Set([0, 1, 2]);

s::has(1);
s::size;
```

<td>

But the extensions system’s special variable namespace is **very contentious**,
as illustrated by its [2020-11 meeting notes][].
Several committee members have signaled that
they will probably block any proposal
that introduces such a new special variable namespace.

In contrast, the bind-`this` operator would involve **no** special variable namespace.
Although this requires more **repetition**,
it also allows the developer to be **explicit** (more [syntactic salt][])
and to define a namespace object in the **ordinary** way.

```js
const $ = {
  has: getIntrinsic('Set.prototype.has'),
  add: getIntrinsic('Set.prototype.add'),
  { get: getSize }:
    getIntrinsic('Set.prototype.size'),
};
const s = new Set([0, 1, 2]);
s->($.has)(1);
s->($.getSize)();
```

<tr>
<td>

It is true that, sometimes, identifiers have **ambiguous** names.

In this example, the homonymous “map”s refer to both a **verb** and a **noun**.
If they have truly identical identifiers, then [shadowing][] will occur.

The verb “map” is distinguished from the noun “map”
by being in the special variable namespace.

```js
// ::map is verb
const ::{ map } from Array;

// map is noun
let map = new Map();

function foo (arr) {
  // ::map is verb
  arr::map(f);
}
```

<td>

But the special variable namespace does not provide much **additional** benefit to developers.
There are **many** words like this in English and other human languages,
not just for the names of extension methods.<br>
JavaScript developers **already** work around linguistic ambiguity all the time
by being **careful** with their names – or by judiciously using **namespace objects**.

```js
// $map is verb
const { map as $map } from Array;

// map is noun
let map = new Map();

function foo() {
  // $map is verb
  a->$map(f);
}
```

<tr>
<td>

The extensions system also adds a **ternary operator**
that allows referring inline to properties
from a specific **constructor** object’s prototype,
rather than variables from the current lexical scope’s special variable namespace.

```js
s::Set:has(1);
// This is equivalent:
const ::has = Set.has;
s::has(1);
```

<td>

This is similar to how the bind-`this` operator
allows its right-hand operand to be an arbitrary expression,
as long as it evaluates into a function.
Because it may be an arbitrary expression,
it is both more verbose, more explicit, and more flexible.

```js
s->(Set.has)(1);
// This is equivalent:
const $has = Set.has;
s->$has(1);
```

<tr>
<td>

The extensions system envisions developers
frequently extracting quasi-[extension methods][]
from built-in prototypes.
It is for this reason that the system’s ternary operator
implicitly extracts prototype methods from constructor objects.

```js
indexed::Array:map(x => x * x);
indexed::Array:filter(x => x > 0);

const ::{
  map, filter,
} from Array;

indexed::map(x => x * x);
indexed::filter(x => x > 0);
```

<td>

This use case is made clearer with **explicit** code
that explicitly accesses or extracts properties from the prototypes:
serving as more [syntactic salt][].

The bind-`this` operator would encourage such clarity and explicitness.

```js
indexed->(Array.prototype.map)(x => x * x);
indexed->(Array.prototype.filter)(x => x > 0);

const {
  map: $map, filter: $filter,
} = Array.prototype;

indexed->$map(x => x * x);
indexed->$filter(x => x > 0);
```

<tr>
<td>

The extensions system’s ternary operator,
like the `import`-like “destructuring” syntax
`const ::{ … } from …;`,
is also **polymorphic**.
The ternary operator changes depending on whether its middle operand
is a **constructor** or a non-constructor (i.e., **“namespace”**) object.

```js
import * as _ from 'lodash';
[0, 1, 2]::_:take(2);
// This is equivalent to
// _.take([0, 1, 2], 2).
```

If the middle operand is **not** a constructor,
then it calls its right-hand side as a static method
on the middle operand, with the left-hand side as its first argument.

In the previous two examples, `Set` and `Array` were constructors,
but in this example, `_` is a non-constructor namespace object.

<td>

This static-method-calling functionality
also overlaps with the [pipe operator][].

```js
import * as _ from 'lodash';
[0, 1, 2] |> _.take(^, 2);
// This is equivalent to
// _.take([0, 1, 2], 2).
```

The pipe operator is more versatile,
being able to work with any expression.

<tr>
<td>

Because `WebAssembly` is not a constructor,
the Eetensions ternary operator calls a static method.

```js
fetch('simple.wasm')
  ::WebAssembly:instantiateStreaming();
```

<td>

But the pipe operator also would also linearize this nested expression.

```js
fetch('simple.wasm')
  |> WebAssembly.instantiateStreaming(^);
```

<tr>
<td>

Because of the **polymorphism** of the extensions system’s ternary operator
(which is based on whether its middle operand is a constructor or not),
and because `Object` is a **constructor**,
the **first** statement means `Object.prototype.toString.call(value)`.
In contrast, we cannot use the ternary operator
for the **second** statement’s `Object.keys` **static** method –
because `Object` is a constructor.

```js
obj::Object:toString();
Object.keys(obj);
```

<td>

The bind-`this` operator would **not** attempt to improve **both**
applying `Object`’s prototype methods **and** applying `Object`’s static methods.
Instead, it only solves applying `Object`’s prototype methods
(while requiring the developer to be **explicit**
about their extraction from `Object.prototype` – more [syntactic salt][]).

If we want to convert a static-method call into a postfix chaining form,
we can use the [pipe operator][] again.

```js
obj->(Object.prototype.toString)();
obj |> Object.keys(^);
```

<tr>
<td>

The extensions system can work with
the [symbol-based **protocols** proposal][protocols].

The following example assumes that `iter.constructor`
implements some Foldable protocol,
whose properties are keyed with the **symbols**
`Foldable.toArray` and `Foldable.size`.

The example shows accessing `iter`’s `Foldable` properties
using a “**qualified** form” (ternary operator)
and an “**unqualified** form” (binary operator).

```js
// Qualified form.
iter::Foldable:toArray();
iter::Foldable:size;

// Unqualified form.
const ::{
  toArray,
  size,
} from Foldable;
iter::toArray();
iter::size;
```

<td>

However, there is not that much additional benefit to the developer.
`Foldable.toArray` and `Foldable.size`
**already** work with the ordinary property-access `[]` notation:
a **qualified** form without the Extensions ternary operator.
Likewise, assigning the symbol keys to ordinary variables,
then using them with ordinary property access,
gives an **unqualified** form, without needing the extensions binary operator.

It is true that the developer has to be careful
with their symbol variables’ names.
This is **nothing new**:
developers **already** have to be careful with their variables’ names,
and they have **already** developed **their own systems**.

```js
// Qualified form.
iter[Foldable.toArray]();
iter[Foldable.size];

// Unqualified form.
const {
  toArray: $toArray,
  size: $size,
} = Foldable;
iter[$toArray]();
iter[$size];
```

<tr>
<td>

The extensions system could serve as a replacement for
the proposed **[extended numerics][]** system.

```js
1::px + 3::px;
1::CSS:px + 3::CSS:px;
```

<td>

But this unfortunately does not save **any** characters
over ordinary function calls.

```js
px(1) + px(3);
CSS.px(1) + CSS.px(3);
```

<tr>
<td>

The extensions system envisions **library** developers
writing new functions within the **special** variable **namespace**.
A **special `import`** form would be required for these functions.

```js
// Module `foo`
export const ::at = function (i) {
  return this[
    i >= 0
    ? i
    : this.length + i
  ];
};

// Another module
import ::{ bindKey } from 'foo';

'Hello world'.split(' ')
  ::at(0).toUpperCase()
  ::at(-1);
```
This may cause **ecosystem schism**
between libraries that use the special variable namespace
and libraries that use the ordinary variable namespace.

<td>

In contrast, because the bind-`this` operator
involves **no** special variable namespace.

```js
// Module `foo`
export const at = function (i) {
  return this[
    i >= 0
    ? i
    : this.length + i
  ];
};

// Another module
import $at from 'foo';
'Hello world'.split(' ')
  ->$at(0).toUpperCase()
  ->$at(-1);
```
Libraries may export only to the single ordinary variable namespace.
There is therefore little risk of ecosystem schism.

<tr>
<td>

The extensions system’s ternary operator
is further **customizable** with a `Symbol.extension` property.

```js
const ::extract = {
  [Symbol.extension]: {
    get (target, key) {
      return target[key].bind(target);
    },
  },
};
const user = {
  name: 'hax',
  greet () { return `Hi ${this.name}!`; }
};
const f = user::extract:greet;
f(); // 'Hi hax!'
```

The value of the `Symbol.extension` property
may be a “description” object with `get`, `set`, and/or `invoke` methods.

In this way, the extensions system’s ternary operator thus is actually a **metaprogramming** tool.

<td>


The narrowly scoped bind-`this` operator does not attempt to provide this same metaprogramming functionality.

```js
const user = {
  name: 'hax',
  greet () { return `Hi ${this.name}!`; }
};
const f = user->greet;
f(); // 'Hi hax!'
```

<tr>
<td>

The metaprogramming of the extensions’ ternary operator
can customize any getting, setting, and/or invocation
of its extension methods.
This metaprogramming can make the proposed [eventual send][] more convenient.
(The following code uses seperately defined
[`send` and `sendOnly` extension namespace objects][send extension].)

```js
const fileP = target
  ::send:openDirectory(dirName)
  ::send:openFile(fileName);
const contents = await fileP::send:read();
console.log('file contents', contents);
fileP::sendOnly:append('fire-and-forget');
```

<td>

However, this metaprogramming overlaps with **proxy objects**,
which provide similar capabilities.

For example, [eventual send][] uses proxies to give
its customized getting/setting/invocation behavior.

When we need to wrap objects in proxies repeatedly,
we can use the [pipe operator][].

```js
const fileP = target
  |> E(^).openDirectory(dirName)
  |> E(^).openFile(fileName);
const contents = await E(fileP).read();
console.log('file contents', contents);
fileP |> E.sendOnly(^).append('fire-and-forget');
```

</table>

***

The [extensions system][] is an **ambitious** metaprogramming proposal
that attempts to solve several different problems in a unified fashion.
Its logic is thus:

1. “Being able to extract/bind and call **methods** is **important**
   but **clunky**. We should improve their ergonomics with syntax.”
2. “Extracting **get/set accessors** is also important, so it should get **syntax too**.”
3. “And the syntax for extracting get accessors should look **similar**
   to the syntax for extracting methods.” (Hence, its import-like “destructuring” syntax.)
4. “And the syntax for extracting, importing, and using these should make it **difficult**
   to cause **[name collision][shadowing]** with other variables.”
   (Hence the special variable namespace.)
5. “It would also be good if the syntax could have **extendable behavior**.”
   (Hence, metaprogramming with `Symbol.extensions`.)

***

However, several of these points are debatable.

1. “Being able to extract/bind and call methods is important
   but clunky. We should improve their ergonomics with syntax.”

   Both proposals agree with this statement.
   [`.bind`, `.call`, and `.apply` are very common][common],
   but [they are also very clunky][clunky].

2. “Extracting get/set accessors is also important, so it should get syntax too.”

   This is **debatable**. Unlike extracting methods,
   extracting get accessors is **uncommon**.

3. “And the syntax for extracting get accessors should look similar
   to the syntax for extracting methods.” (Hence, its import-like “destructuring” syntax.)

   This is **very debatable**. Get/set **accessors** are **not** the same as **methods**.
   Methods are **properties** that happen to be functions.
   Accessors are **not** properties;
   they are **functions** that activate when getting or setting properties.

4. “And the syntax for extracting, importing, and using these should make it difficult
   to cause [name collision][shadowing] with other variables.”
   (Hence the special variable namespace.)

   This is **very controversial**. Programmers **already** deal with name ambiguity
   and variable shadowing all the time with **their own naming systems**.
   A new language namespace is **cognitively heavy**, is complex for **implementors**,
   decreases **interoperability**, and may cause an **ecosystem schism**.
   Several TC39 members, such as Waldemar Horwat and Jordan Halbard,
   are **strongly against any** new syntactic namespace for identifiers
   (see [2020-11 meeting notes][]).

5. “It would also be good if the syntax could have extendable behavior.”
   (Hence, metaprogramming with `Symbol.extensions`.)

   Although the goal of solving multiple proposals’ problems
   with a single metaprogramming system is **laudable**,
   if the **foundation** it is built is **not viable**,
   then the metaprogramming system is not viable either.

***

In contrast, the [bind-`this` operator][bind-this] is focused on **one problem**:

1. [`.bind`][bind], [`.call`][call], and [`.apply`][apply]
   are very **useful** and very **common** in JavaScript codebases…
2. …but `.bind`, `.call`, and `.apply` are **clunky** and **unergonomic**.

All the other issues that the extensions system solves
is either solved by the [pipe operator][] –
or **already** solved with existing features
such as ordinary variables, namespace objects, proxies.

***

The [extensions system][] is an ambitious and laudable exploration.
However, it has little hope of advancing in TC39 as it is.
And it tries to solve more problems than necessary.

But there is still a real need for a syntax
that makes [`.bind`][bind], [`.call`][call], and [`.apply`][apply]
less clunky and more ergonomic.

A single, simple [bind-`this` operator][bind-this] without extra features
is much more likely to reach total TC39 consensus,
would be easier for developers to reason about,
and does not carry a risk of ecosystem schism.

[extensions system]: https://github.com/tc39/proposal-extensions
[bind-this]: https://github.com/js-choi/proposal-bind-operator
[syntactic salt]: https://en.wikipedia.org/wiki/Syntactic_sugar#Syntactic_salt
[getIntrinsic]: https://github.com/ljharb/proposal-get-intrinsic
[pipe operator]: https://github.com/js-choi/proposal-hack-pipes
[extension methods]: https://en.wikipedia.org/wiki/Extension_method
[shadowing]: https://en.wikipedia.org/wiki/Variable_shadowing
[protocols]: https://github.com/tc39/proposal-first-class-protocols
[extended numerics]: https://github.com/tc39/proposal-extended-numeric-literals/
[eventual send]: https://github.com/tc39/proposal-eventual-send
[send extension]: https://johnhax.net/2020/tc39-nov-ext/slide#124
[bindKey]: https://lodash.com/docs/4.17.15#bindKey
[map]: https://lodash.com/docs/4.17.15#map
[2020-11 meeting notes]: https://github.com/tc39/notes/blob/master/meetings/2020-11/nov-19.md#extensions-for-stage-1
[security use case]: https://github.com/js-choi/proposal-bind-this/blob/main/security-use-case.md
[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call
[apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply
[common]: https://github.com/js-choi/proposal-bind-this/blob/main/README.md#bind-call-and-apply-are-very-common
[clunky]: https://github.com/js-choi/proposal-bind-this/blob/main/README.md#bind-call-and-apply-are-clunky
