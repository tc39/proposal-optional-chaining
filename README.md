# Optional Chaining for JavaScript

## Status
Current Stage:
* Stage 1

## Authors
* Claude Pache (@claudepache)
* Gabriel Isenberg (@the_gisenberg)

## Overview and motivation
When looking for a property value deeply in a tree structure, one has often to check whether intermediate nodes exist:

```javascript
var street = user.address && user.address.street;
```

Also, many API return either an object or null/undefined, and one may want to extract a property from the result only when it is not null:

```javascript
var fooInput = myForm.querySelector('input[name=foo]')
var fooValue = fooInput ? fooInput.value : undefined
```

The Optional Chaining Operator allows a developer to handle many of those cases without repeating themselves and/or assigning intermediate results in temporary variables:

```
var street = user.address?.street
var fooValue = myForm.querySelector('input[name=foo]')?.value
```

## Prior Art
* C#: [Null-conditional operator](https://msdn.microsoft.com/en-us/library/dn986595.aspx)
* Swift: [Optionals](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID330)
* Groovy: [Safe navigation operator](http://groovy-lang.org/operators.html#_safe_navigation_operator)
* Ruby: [Safe navigation operator](http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/)
* CoffeeScript: [Existential operator](http://coffeescript.org/#existential-operator)

## Syntax

Optional property navigation
```
obj?.prop       // optional property access
obj?.[expr]     // optional property access
func?.(...args) // optional function or method call
```

### Notes
* In order to allow `foo?.3:0` to be parsed as `foo ? .3 : 0` (as required for backward compatibility), a simple lookahead is added at the level of the lexical grammar, so that the sequence of characters `?.` is not interpreted as a single token in that situation (the `?.` token must not be immediately followed by a decimal digit).

## Semantics
*Base case*. If the expression at the left-hand side of the `?.` operator evaluates to undefined or null, its right-hand side is not evaluated and the whole expression returns undefined.

```
a?.b         // undefined if a is null/undefined, a.b otherwise
a?.[++x]     // If a evaluates to null/undefined, the variable x is *not* incremented.
a?.b.c().d   // undefined if a is null/undefined
             // throws if b or c is null/undefined
             // throws if c() returns null/undefined

a?.()        // a is invoked if not null/undefined
```

## TODO
Per the [TC39 process document](https://tc39.github.io/process-document/), here is a high level list of work that needs to happen across the various proposal stages.

* [x] Identify champion to advance addition (stage-1)
* [x] Prose outlining the problem or need and general shape of the solution (stage-1)
* [x] Illustrative examples of usage (stage-1)
* [x] High-level API (stage-1)
* [x] [Initial spec text](https://claudepache.github.io/es-optional-chaining/) (stage-2)
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
