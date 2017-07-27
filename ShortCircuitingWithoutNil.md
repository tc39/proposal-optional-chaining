This is an attempt to rewrite Optional Chaining without using a transient Nil reference for implementing short-circuiting semantics.

With this version:
* “optional construction” as in `new foo?.()` is abandoned;
* constructions like `new a?.b` or ``a?.b `{c}` `` are not allowed by the grammar;
* parentheses as in `(a?.b).c` stop short-circuiting.

**Note.** A previous version of this document rewrote MemberExpression and CallExpression productions in order to be consistent with how OptionalChainingExpression is specified below. We removed that part, because: 

* the consistency is incomplete, as there are still separate MemberExpression, CoverCallExpressionAndAsyncArrowHead and CallExpression productions for various technical reasons, and that leads to break a chain of property accesses, method calls, etc. in several parts;
* until otherwise proven, we don’t need to change what works.


What is important, is to ensure consistency of the observed behaviour (for that, factoring the relevant part of algorithms may help).


Grammar
=======

In the lexical grammar
```
      OptionalChainingOperator ::
        ? . [lookahead ∉ DecimalDigit]
```
Then, adding to the syntax of [Left-Hand-Side Expressions](https://tc39.github.io/ecma262/#sec-left-hand-side-expressions):
```
OptionalAccessChain :
    OptionalChainingOperator [ Expression ]
    OptionalChainingOperator IdentifierName
    OptionalChainingOperator Arguments
    OptionalAccessChain [ Expression ]
    OptionalAccessChain . IdentifierName
    OptionalAccessChain Arguments
```
Example of OptionalAccessChain:  ``?.a[b].c(d)``
```
OptionalChainingExpression :
    MemberExpression OptionalAccessChain
    CallExpression OptionalAccessChain
    OptionalChainingExpression OptionalAccessChain
```

Finally, the current `LeftHandSideExpression` is replaced with:
```
NonOptionalChainingExpression :
    NewExpression
    CallExpression
    
LeftHandSideExpression :
    NonOptionalChainingExpression
    OptionalChainingExpression
```
The distinction between `NonOptionalChainingExpression` and `OptionalChainingExpression` is useful if/when we want to spec
optional assignment. (For optional deletion, nothing is needed, not even touching a jot in the relevant part of the spec.)


Runtime semantics
=================
Shorthand notations
-------------------
   * “someNode.SomeOperation(foo, bar)” means “SomeOperation of someNode with arguments foo and bar”;
   * “someNode.Evaluation()” means “the result of evaluating someNode”.

Evaluation
----------

```
OptionalChainingExpression :
    MemberExpression OptionalAccessChain
    CallExpression OptionalAccessChain
    OptionalChainingExpression OptionalAccessChain
``` 
1. Let baseExpression be the first child of this production (i.e., this MemberExpression, CallExpression, or OptionalChainingExpression).
1. Let ref be ? baseExpression.Evaluation().
1. Let val be ? GetValue(ref).
1. If Type(val) is Null or Undefined, then
    1. Return undefined.
1. Return ? OptionalAccessChain.ChainEvaluation(ref, val).


ChainEvaluation(ref, val)
-----------------------
This operation is defined for OptionalAccessChain productions.

```
OptionalAccessChain :
    OptionalChainingOperator [ Expression ]
```    
See https://tc39.github.io/ecma262/#sec-property-accessors-runtime-semantics-evaluation, omitting the two first steps,
and replacing baseReference and baseValue with ref and val respectively.

```
OptionalAccessChain :
    OptionalAccessChain OptionalChainingOperator [ Expression ]
```    

1. Let firstNode be the first child of this production.
1. Let ref2 be ? firstNode.ChainEvaluation(ref, val).
1. Let val2 be ? GetValue(ref2).
1. Continue as in https://tc39.github.io/ecma262/#sec-property-accessors-runtime-semantics-evaluation, omitting the two first steps,
and replacing baseReference and baseValue with ref2 and val2 respectively.

```
OptionalAccessChain :
    OptionalAccessChain Arguments
```    
1. Let thisCallAccess be the parse of the production.
1. Let tailCall be IsInTailPosition(thisCallAccess).
1. Return ? EvaluateCall2(ref, val, Arguments, tailCall).

Here, EvaluateCall2(ref, func, Arguments, tailCall) is [EvaluateCall(ref, Arguments, tailCall)](https://tc39.github.io/ecma262/#sec-evaluatecall) with first step removed.

Other algorithms are left as exercise to the reader.
