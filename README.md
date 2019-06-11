# Optional Chaining for JavaScript

## Status
Current Stage:
* Stage 2

## Authors
* Claude Pache (@claudepache)
* Gabriel Isenberg (@the_gisenberg)
* Dustin Savery (@dustinsavery)

## Overview and motivation
When looking for a property value that's deep in a tree-like structure, one often has to check whether intermediate nodes exist:

```javascript
var street = user.address && user.address.street;
```

Also, many API return either an object or null/undefined, and one may want to extract a property from the result only when it is not null:

```javascript
var fooInput = myForm.querySelector('input[name=foo]')
var fooValue = fooInput ? fooInput.value : undefined
```

The Optional Chaining Operator allows a developer to handle many of those cases without repeating themselves and/or assigning intermediate results in temporary variables:

```javascript
var street = user.address?.street
var fooValue = myForm.querySelector('input[name=foo]')?.value
```

When some other value than `undefined` is desired for the missing case, this can usually be handled with the [Nullish coalescing operator](//github.com/tc39/proposal-nullish-coalescing):

```javascript
// falls back to a default value when response.setting is missing or nullish
// (response.settings == null) or when respsonse.setting.animationDuration is missing
// or nullish (response.settings.animationDuration == null)
const animationDuration = response.settings?.animationDuration ?? 300;

```

The call variant of Optional Chaining is useful for dealing with interfaces that have optional methods:

```js
iterator.return?.() // manually close an iterator
```
or with methods not universally implemented:
```js
if (myForm.checkValidity?.() === false) { // skip the test in older web browsers
    // form validation fails
    return;
}
```

## Prior Art
Unless otherwise noted, in the following languages, the syntax consists of a question mark prepending the operator, (`a?.b`, `a?.b()`, `a?[b]` or `a?(b)` when applicable).

The following languages implement the operator with the same general semantics as this proposal (i.e., 1) guarding against a null base value, and 2) short-circuiting application to the whole chain):
* C#: [Null-conditional operator](https://msdn.microsoft.com/en-us/library/dn986595.aspx) — null-conditional member access or index, in read access.
* Swift: [Optional Chaining](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/OptionalChaining.html#//apple_ref/doc/uid/TP40014097-CH21-ID245) — optional property, method, or subscript call, in read and write access.
* CoffeeScript: [Existential operator](http://coffeescript.org/#existential-operator) — existential operator variant for property accessor, function call, object construction (`new a?()`). Also applies to assignment and deletion.

The following languages have a similar feature, but do not short-circuit the whole chain when it is longer than one element. This is justified by the fact that, in those languages, methods or properties might be legitimatally used on null (e.g., null.toString() == "null" in Dart):
* Kotlin: [Safe calls](https://kotlinlang.org/docs/reference/null-safety.html#safe-calls) — optional property access for read; optional property assignment for write.
* Dart: [Conditional member access](https://dart.dev/guides/language/language-tour#other-operators) — optional property access.
* Ruby: [Safe navigation operator](https://ruby-doc.org/core-2.6/doc/syntax/calling_methods_rdoc.html#label-Safe+navigation+operator) — Spelled as: `a&.b`


The following languages have a similar feature. We haven’t checked whether they have significant differences in semantics with this proposal:
* Groovy: [Safe navigation operator](http://groovy-lang.org/operators.html#_safe_navigation_operator)

## Syntax

The Optional Chaining operator is spelled `?.`. It may appear in three positions:
```javascript
obj?.prop       // optional static property access
obj?.[expr]     // optional dynamic property access
func?.(...args) // optional function or method call
```

### Notes
* In order to allow `foo?.3:0` to be parsed as `foo ? .3 : 0` (as required for backward compatibility), a simple lookahead is added at the level of the lexical grammar, so that the sequence of characters `?.` is not interpreted as a single token in that situation (the `?.` token must not be immediately followed by a decimal digit).

## Semantics

### Base case
If the operand at the left-hand side of the `?.` operator evaluates to undefined or null, the expression evaluates to undefined. Otherwise the targeted property access, method or function call is triggered normally.

Here are basic examples, each one followed by its desugaring. (The desugaring is not exact in the sense that the LHS should be evaluated only once.)
```js
a?.b                          // undefined if `a` is null/undefined, `a.b` otherwise.
a == null ? undefined : a.b

a?.[x]                        // undefined if `a` is null/undefined, `a[x]` otherwise.
a == null ? undefined : a[x]

a?.b()                        // undefined if `a` is null/undefined
a == null ? undefined : a.b() // throws a TypeError if `a.b` is not a function
                              // otherwise, evaluates to `a.b()`

a?.()                        // undefined if `a` is null/undefined
a == null ? undefined : a()  // throws a TypeError if `a` is neither null/undefined, nor a function
                             // invokes the function `a` otherwise
```

### Short-circuiting

If the expression on the LHS of `?.` evaluates to null/undefined, the RHS is not evaluated. This concept is called *short-circuiting*.

```js
a?.[++x]         // `x` is incremented if and only if `a` is not null/undefined
a == null ? undefined : a[++x]
```

### Long short-circuiting

In fact, short-circuiting, when triggered, skips not only the current property access, method or function call, but also the whole chain of property accesses, method or function calls directly following the Optional Chaining operator.

```js
a?.b.c(++x).d  // if `a` is null/undefined, evaluates to undefined. Variable `x` is not incremented.
               // otherwise, evaluates to `a.b.c(++x).d`.
a == null ? undefined : a.b.c(++x).d
```

Note that the check for nullity is made on `a` only. If, for example, `a` is not null, but `a.b` is null, a TypeError will be thrown when attempting to access the property `"c"` of `a.b`.

This feature is implemented by, e.g., C# and CoffeeScript [TODO: provide precise references].

### Stacking

Let’s call *Optional Chain* an Optional Chaining operator followed by a chain of property accesses, method or function calls.

An Optional Chain may be followed by another Optional Chain.

```js
a?.b[3].c?.(x).d
a == null ? undefined : a.b[3].c == null ? undefined : a.b[3].c(x).d
  // (as always, except that `a` and `a.b[3].c` are evaluated only once)
```

### Edge case: grouping

Parentheses limit the scope of short-circuiting:

```js
(a?.b).c
(a == null ? undefined : a.b).c
```

That follows from the design choice of specifying the scope of short-circuiting by syntax (like the `&&` operator), rather than propagation of a Completion (like the `break` instruction) or an adhoc Reference (like an [earlier version of this proposal](https://github.com/claudepache/es-optional-chaining)). In general, syntax cannot be arbitrarly split by parentheses: for example, `({x}) = y` is not destructuring assignment, but an attempt to assign a value to an object literal.

Note that, whatever the semantics are, there is no practical reason to use parentheses in that position anyway.

### Optional deletion

Because the `delete` operator is very liberal in what it accepts, we have that feature for free:
```js
delete a?.b
a == null ? true : delete a.b
```
where `true` is the usual result of attempting to delete a non-Reference.

<details><summary>Why do we support optional deletion?</summary>

*   **laziness** (this argument is placed first, not because it is the most important, but because it puts the other ones in the right perspective). Laziness (together with impatience and hubris) is one of the most important virtues of the spec writer. That is, all other things being almost equal, take the solution that involves less stuff to be incorporated in the spec.

    Now, it happens that _supporting_ optional deletion requires literally zero effort, while _not supporting_ it (by making the construct an early syntax error) requires some nontrivial effort. (See [PR #73 (comment)](https://github.com/tc39/proposal-optional-chaining/pull/73#issuecomment-442762076) for technical details.)

    Thus, per the laziness principle, the right question is not: “Are there good reasons to support optional deletion?”, but rather: “Are there good reasons to _remove_ support for optional deletion?”

*   **lack of strong reason** for _removing_ support. The supported semantics of optional deletion is the only one that could be expected (provided that the semantics of the delete operator is correctly understood, of course). It is not like it could in some manner confuse the programmer. In fact, the only real reason is: “We didn’t intend to support it.”

*   **consistency of the delete operator**. It is a fact of life that this operator is very liberal in what it accepts, even pretending to succeed when it is given something that does not make sense (e.g., `delete foo()`). The only error conditions (early or not) are in strict mode, when attempting to delete something that is barred from deletion (nonconfigurable property, variable, ...). Supporting optional deletion fits well in that model, while forbidding it does not.
</details>

## Not supported

Although they could be included for completeness, the following are not supported due to lack of real-world use cases or other compelling reasons; see [Issue # 22](https://github.com/tc39/proposal-optional-chaining/issues/22) and [Issue #54](https://github.com/tc39/proposal-optional-chaining/issues/54) for discussion:

* optional construction: `new a?.()`
* optional template literal: ``a?.`string` ``
* constructor or template literals in/after an Optional Chain: `new a?.b()`, ``a?.b`string` ``

The following is not supported, although it has some use cases; see [Issue #18](//github.com/tc39/proposal-optional-chaining/issues/18) for discussion:

* optional property assignment: `a?.b = c`

All the above cases will be forbidden by the grammar or by static semantics so that support might be added later.

## Out of scope

There has been various interesting ideas for applying the idea of “optional” to other constructs. However, there are not part of this proposal. For example:

* optional spread, see [Issue #55](//github.com/tc39/proposal-optional-chaining/issues/55);
* optional destructuring, see [Issue #74](//github.com/tc39/proposal-optional-chaining/issues/74).

## Open issues

### Private class fields and methods
[Issue #28](https://github.com/tc39/proposal-optional-chaining/issues/28): Should optional chains support the upcoming private [class fields](https://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods), as in `a?.#b`, `a?.#b()` or `a?.b.#c`? Quoting [microsoft/TypeScript#30167 (comment)](https://github.com/microsoft/TypeScript/issues/30167#issuecomment-468881537):

> This one isn't baked into the proposal yet, simply because private fields themselves aren't baked yet. So we don't want to hold up this proposal if that one happens to stall out. Once that one has reached Stage 4, we will address it then.


## FAQ


<dl>


<dt>obj?.[expr]  and  func?.(arg)  look ugly. Why not use  obj?[expr]  and  func?(arg)  as does &lt;language X>?

<dd>

We don’t use the `obj?[expr]` and `func?(arg)` syntax, because of the difficulty for the parser to efficiently distinguish those forms from the conditional operator, e.g., `obj?[expr].filter(fun):0` and `func?(x - 2) + 3 :1`.

Alternative syntaxes for those two cases each have their own flaws; and deciding which one looks the least bad is mostly a question of personal taste. Here is how we made our choice:

* pick the best syntax for the `obj?.prop` case, which is expected to occur most often;
* extend the use of the recognisable `?.` sequence of characters to other cases: `obj?.[expr]`, `func?.(arg)`.

As for &lt;language X>, it has different syntactical constraints than JavaScript because of &lt;some construct not supported by X or working differently in X>.

<dt>Ok, but I <b>really</b> think that <i>&lt;alternative syntax></i> is better
  
<dd>
  
Various alternative syntaxes has been explored and extensively discussed in the past. None of them gained consensus. Search for [issues
with label “alternative syntax”](https://github.com/tc39/proposal-optional-chaining/issues?utf8=%E2%9C%93&q=label%3A%22alternative+syntax%22), as well as [issues
with label “alternative syntax and semantics”](https://github.com/tc39/proposal-optional-chaining/issues?utf8=%E2%9C%93&q=label%3A%22alternative+syntax+and+semantics%22) for those that had impact on semantics.

<dt>Why does (null)?.b evaluate to undefined rather than null?

<dd>

Neither `a.b` nor `a?.b` is intended to preserve arbitrary information on the base object `a`, but only to give information about the property `"b"` of that object. If a property `"b"` is absent from `a`, this is reflected by `a.b === undefined` and `a?.b === undefined`.

In particular, the value `null` is considered to have no properties; therefore, `(null)?.b` is undefined.



<dt>Why do you want long short-circuiting?</dt>

<dd>

See [Issue #3 (comment)](https://github.com/tc39/proposal-optional-chaining/issues/3#issuecomment-306791812).



<dt>In a?.b.c, if a.b is null, then a.b.c will evaluate to undefined, right?

<dd>

No. It will throw a TypeError when attempting to fetch the property `"c"` of `a.b`.

The opportunity of short-circuiting happens only at one time, just after having evaluated the LHS of the Optional Chaining operator. If the result of that check is negative, evaluation proceeds normally.

In other words, the `?.` operator has an effect only at the very moment it is evaluated. It does not change the semantics of subsequent property accesses, method or function calls.



<dt>In a deeply nested chain like `a?.b?.c`, why should I write `?.` at each level? Should I not be able to write the operator only once for the whole chain?</dt>

<dd>

By design, we want the developer to be able to mark each place that they expect to be null/undefined, and only those. Indeed, we believe that an unexpected null/undefined value, being a symptom of a probable bug, should be reported as a TypeError rather than swept under the rug.


<dt>... but, in the case of a deeply nested chain, we almost always want to test for null/undefined at each level, no?

<dd>

Deeply nested tree-like structures is not the sole use case of Optional Chaining.

See also [Usage statistics on optional chaining in CoffeeScript](https://github.com/tc39/proposal-optional-chaining/issues/17) and compare “Total soak operations” with “Total soak operations chained on top of another soak”.

<dt>The feature looks like an error suppression operator, right?
  
<dd>

No. Optional Chaining just checks whether some value is undefined or null. It does not catch or suppress errors that are thrown by evaluating the surrounding code. For example:

```js
(function () {
    "use strict"
    undeclared_var?.b    // ReferenceError: undeclared_var is not defined
    arguments?.callee    // TypeError: 'callee' may not be accessed in strict mode
    arguments.callee?.() // TypeError: 'callee' may not be accessed in strict mode
})()
```

</dl>

## Specification
See: https://tc39.github.io/proposal-optional-chaining/


## TODO
Per the [TC39 process document](https://tc39.github.io/process-document/), here is a high level list of work that needs to happen across the various proposal stages.

* [x] Identify champion to advance addition (stage-1)
* [x] Prose outlining the problem or need and general shape of the solution (stage-1)
* [x] Illustrative examples of usage (stage-1)
* [x] High-level API (stage-1)
* [x] [Initial spec text](https://tc39.github.io/proposal-optional-chaining/) (stage-2)
* [x] [Babel plugin](https://github.com/babel/babel/pull/5813) (stage-2)
* [ ] Finalize and reviewer signoff for spec text (stage-3)
* [ ] Test262 acceptance tests (stage-4)
* [ ] tc39/ecma262 pull request with integrated spec text (stage-4)
* [ ] Reviewer signoff (stage-4)

## References
* [TC39 Slide Deck: Null Propagation Operator](https://docs.google.com/presentation/d/11O_wIBBbZgE1bMVRJI8kGnmC6dWCBOwutbN9SWOK0fU/edit?usp=sharing)
* [es-optional-chaining](https://github.com/claudepache/es-optional-chaining) (@claudepache)
*  [ecmascript-optionals-proposal](https://github.com/davidyaha/ecmascript-optionals-proposal) (@davidyaha)

## Related issues
* [Babylon implementation](https://github.com/babel/babylon/issues/328)
* [estree: Null Propagation Operator](https://github.com/estree/estree/issues/146)
* [TypeScript: Suggestion: "safe navigation operator"](https://github.com/Microsoft/TypeScript/issues/16)
* [Flow: Syntax for handling nullable variables](https://github.com/facebook/flow/issues/607)

## Prior discussion
* https://esdiscuss.org/topic/existential-operator-null-propagation-operator
* https://esdiscuss.org/topic/optional-chaining-aka-existential-operator-null-propagation
* https://esdiscuss.org/topic/specifying-the-existential-operator-using-abrupt-completion

