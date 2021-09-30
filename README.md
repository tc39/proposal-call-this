# Bind-`this` operator for JavaScript
ECMAScript Stage-0 Proposal. J. S. Choi, 2021.

* **[Formal specification][]**
* Babel plugin: Not yet

[formal specification]: http://jschoi.org/21/es-bind-operator/

## Description
(A [formal specification][] is available.)

**Method binding** `->` is a **left-associative infix operator**.
Its right-hand side is a **chain of identifiers identifier** (like `f` or `o.x.y`)
or an **expression** in `(` `)` (like `hof()`),
either of which must evaluate to a **function**.
Its left-hand side is some expression that evaluates to an **object**.
The `->` operator **binds** its left-hand side
to its right-hand side’s `this` value,
creating a **bound function** in the same manner
as [`Function.prototype.bind`][bind].

For example, `arr->fn` would be roughly
equivalent to `fn.bind(arr)`,
except that its behavior does **not change**
if code elsewhere **reassigns** the **global method** `Function.prototype.bind`.

The right-hand side may be a property-access chain.
`arr->o.x.y` would be roughly
equivalent to `o.x.y.bind(arr)`.

Likewise, `obj->(createMethod())` would be roughly
equivalent to `createMethod().bind(obj)`.

If the operator’s right-hand side does not evaluate to a function during runtime,
then the program throws a `TypeError`.

Function binding has equal **[precedence][]** with
**member expressions**, call expressions, `new` expressions with arguments,
and optional chains.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                  | Example       |
| ------------------------------- | ------------- |
| Primary expressions             | `a->fn`       |
| Member expressions              | `a.b->fn`     |
| Call expressions                | `a()->fn`     |
|`new` expressions with arguments | `new C()->fn` |
| Optional chains                 | `a?.b->fn`    |

The bound functions created by the bind-`this` operator
are **indistinguishable** from the bound functions
that are already created by [`Function.prototype.bind`][bind].
Both are **exotic objects** that do not have a `prototype` property,
and which may be called like any typical function.

Similarly to the `?.` optional-chaining token,
the `->` token may be **padded by whitespace**.\
For example, `a -> m`\
is equivalent to `a->fn`,\
and `a -> (createFn())`\
is equivalent to `a->(createFn())`.

There are **no other special rules**.

## Why a bind-`this` operator
[`Function.prototype.bind`][call] and [`Function.prototype.call`][bind]
are very common in **object-oriented JavaScript** code.
They are useful methods that allows us to apply functions to any object,
binding their first arguments to the `this` bindings within those functions,
no matter the current object environment.
`bind` and `call` allow us to **extend** an **object** with a function
as if that function were **its own method**.
They serve as an important link between
the **object-oriented** and **functional** styles in JavaScript.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call

Why then would we need an operator that does the same thing?
Because `bind` and `call` are vulnerable to **global mutation**.

For example, when we run our code in an untrusted environment,
an adversary may mutate global prototype objects
such as `Array.prototype`,
reassigning or deleting their methods.

```js
// The adversary’s code.
delete Array.prototype.slice;

// Our own trusted code, running later.
// Due to the adversary, this unexpectedly throws an error.
[0, 1, 2].slice(1, 2);
```

In order to harden JavaScript applications against this attack,
we can extract critical global prototype methods into variables
before any untrusted code may run.
We would then use our critical methods with their `call` methods.

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;

// Our own trusted code, running later.
// In spite of the adversary, this no longer throws an error.
slice.call([0, 1, 2], 1, 2);
```

But this is still vulnerable to mutation of `Function.prototype`:

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;
delete Function.prototype.call;

// Our own trusted code, running later.
// Due to the adversary, this throws an error again.
slice.call([0, 1, 2], 1, 2);
```

There is currently no way to harden code against mutation of `Function.prototype`.
A new operator, however, would not be vulnerable to mutation:

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;
delete Function.prototype.call;

// Our own trusted code, running later.
// In spite of the adversary, this no longer throws an error.
// It also is considerably more readable.
[0, 1, 2]->slice(1, 2);
```

As a bonus, the syntax is also considerably more readable
than code that uses `bind` or `call`.
A bound function could be called inline
as if it were **actually a method** in the object
whose **property key** is the **function itself**:

```js
function extensionMethod () {
  return this;
}

obj.actualMethod();
obj->extensionMethod();
// Compare with extensionMethod.call(obj).
```

The bind-`this` operator can also **extract** a **method** from a **class**
into a function whose first parameter becomes its `this` binding:\
for example, `const { slice } = Array.prototype; arr->slice(1, 3);`.\
It can also similarly extract a method from an **instance**
into a function that always uses that instance as its `this` binding:\
for example, `const arr = arr->(arr.slice); slice(1, 3);`.

## Real-world examples
Only minor formatting changes have been made to the status-quo examples.

<table>
<thead>
<tr>
<th>Status quo
<th>With binding

<tbody>
<tr>
<td>

```js
???
```
From ???.

<td>

```js
???
```

</table>

## Non-goals
A goal of this proposal is **simplicity**.
Therefore, this proposal purposefully
does *not* address the following use cases.

**Tacit method extraction** with another operator
(like `arr&.slice` for `arr.slice.bind(arr.slice)` hypothetically)
would be **nice to have**,
but method extraction is **already possible** with this proposal.\
`const slice = arr->(arr.slice); slice(1, 3);`\
is not much wordier than\
`const slice = arr&.slice; slice(1, 3);`

**Extension getters and setters**
(i.e., extending objects with new property getters or setters
**without mutating** the object)
may **also** be useful,
and this proposal would be **forward compatible** with such a feature
using the **same operator** `->` for **property-object binding**,
in addition to this proposal’s **method binding**.
Getter/setter binding could be added in a separate proposal
using `{ get () {}, set () {} }` objects.
For example, we could add an extension getter `allDivs`
to a `document` object like so:
```js
const allDivs = {
	get () { return this.querySelectorAll('div'); }
};

document->allDivs;
```

**Function/expression application**,
in which **deeply nested** function calls and other expressions
are untangled into **linear pipelines**,
is important but not addressed by this proposal.
Instead, it is addressed by the **pipe operator**,
with which this proposal’s syntax **works well**.\
For example, we could untangle `h(await g(o->f(0, v)), 1)`\
into `v |> o->f(0, %) |> await g(%) |> h(%, 1)`.
