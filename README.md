# Method extraction for JavaScript
ECMAScript Stage-0 Proposal. J. S. Choi, 2021.

* Specification: Not yet
* Babel plugin: Not yet

## Why a method-extraction operator
[`Function.prototype.bind`][bind] is very common in object-oriented JavaScript code.
It is a useful method that allows us to extract object methods
into **bound functions** that are not dependent on a particular `this` value
and which may be used in any object environment.
`bind` serves as an important link between
object-oriented and functional APIs in JavaScript.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind

Why then would an operator that does the same thing?
Because there is an important difference.

### Status quo: `Function.prototype.bind` is insecure
When we run our code in an untrusted environment, global objects may be modified.
In particular, `Function.prototype.call`…

> JHD: Want to delete Function.prototype.call and things still work

> JHD: Because then I'm not relying on the .call API. It's not super common to be robust against things like this, but that doesnt mean its not a good goal. We need to allow users to harden their code and prevent edge cases like this.

> JHD: Defense model here is that you run code in an environment you trust, but after that anything could happen. I use Function.bind.call to protect against this.

### No other current TC39 proposal addresses method extraction
There is a proposal for an extension-method operator `::`.
However, this proposal does not address method extraction as an explicit non-goal,
and it does not overlap with this proposal.

The extension-method proposal is a successor
to an older proposal for a bind operator `::`.
This older proposal did address method extraction, but it is now inactive.

## Description
(A formal draft specification is not yet available.)

**Method extraction** `&.` is a **left-associative infix operator**
that binds its right-hand side (a method identifier
or a dynamic method name in `[` `]`)
to its left-hand side (the method’s original object),
creating a **bound function** in the same manner
as [`Function.prototype.bind`][bind].

For example, `arr&.slice` would be roughly
equivalent to `arr.slice.bind(arr)`,
except that its behavior does not change
if code elsewhere reassigns the global method `Function.prototype.bind`.

Likewise, `obj&.[Symbol.iterator]` would be roughly
equivalent to `obj[Symbol.iterator].bind(obj)`.

Method extraction has equal [precedence][] with
member expressions, call expressions, `new` expressions with arguments,
and optional chains.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                  | Example      |
| ------------------------------- | ------------ |
| Primary expressions             | `a&.m`       |
| Member expressions              | `a.b&.m`     |
| Call expressions                | `a()&.m`     |
|`new` expressions with arguments | `new F()&.m` |
| Optional chains                 | `a?.b&.m`    |

Similarly to the `?.` optional-chaining token,
the `&.` token may be padded by whitespace.
For example, `a &. b`\
is equivalent to `a&.b`,\
and `a &. [Symbol.iterator]`\
is equivalent to `a&.[Symbol.iterator]`.

## Real-world examples
Only minor formatting changes have been made to the status-quo examples.

<table>
<thead>
<tr>
<th>Status quo
<th>With method extraction

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
