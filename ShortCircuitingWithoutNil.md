This is an attempt to rewrite Optional Chaining without using a transient Nil reference for implementing short-circuiting semantics.

With this version:
* “optional construction” as in `new foo?.()` is abandoned;
* constructions like `new a?.b` or ``a?.b `{c}` `` are not allowed by the grammar;
* parentheses as in `(a?.b).c` stop short-circuiting.

Grammar
=======
Replacing https://tc39.github.io/ecma262/#sec-left-hand-side-expressions.

The rewriting of MemberExpression and CallExpression are not strictly needed, but they are here to maintain consistency with how OptionalChainingExpression is specified. 

MemberExpression / NewExpression
--------------------------------
```
MemberAccessChain :
    [ Expression ]
    . IdentifierName
    TemplateLiteral
    MemberAccessChain [ Expression ]
    MemberAccessChain . IdentifierName
    MemberAccessChain TemplateLiteral
```
Example of MemberAccessChain:  ``.a[b].c`{d}` ``
```       
MemberBaseExpression :
    PrimaryExpression
    SuperProperty
    MetaProperty
    new MemberBaseExpression Arguments
    new MemberBaseExpression MemberAccessChain Arguments

MemberExpression :
    MemberBaseExpression
    MemberBaseExpression MemberAccessChain
    
NewExpression :
    MemberExpression
    new NewExpression
```

CallExpression
--------------
```
CallAccessChain :
    [ Expression ]
    . IdentifierName
    TemplateLiteral
    Arguments
    CallAccessChain [ Expression ]
    CallAccessChain . IdentifierName
    CallAccessChain TemplateLiteral
    CallAccessChain Arguments
```
Example of CallAccessChain:  ``.a[b](c,d)`{e}`.f(g)``
```    
CallBaseExpression :
    CoverCallExpressionAndAsyncArrowHead
    SuperCall
    
CallExpression :
    CallBaseExpression
    CallBaseExpression CallAccessChain
```
When processing an instance of the production `CallExpression : CoverCallExpressionAndAsyncArrowHead`
the interpretation of `CoverCallExpressionAndAsyncArrowHead` is refined using the following grammar:
```
CallMemberExpression :
    MemberExpression Arguments
```

OptionalChainingExpression
--------------------------
In the lexical grammar
```
      OptionalChainingOperator ::
        ? . [lookahead ∉ DecimalDigit]
```
Then:
```
OptionalChainingAccessChain :
    OptionalChainingOperator [ Expression ]
    OptionalChainingOperator IdentifierName
    OptionalChainingOperator Arguments
    OptionalChainingAccessChain [ Expression ]
    OptionalChainingAccessChain . IdentifierName
    OptionalChainingAccessChain Arguments
```
Example of OptionalChainingAccessChain:  ``?.a[b].c(d)``
```
OptionalChainingExpression :
    MemberExpression OptionalChainingAccessChain
    CallExpression OptionalChainingAccessChain
    OptionalChainingExpression OptionalChainingAccessChain
```
Note the recursion in the definition of `OptionalChainingExpression`,
which is not used/needed for MemberExpression or CallExpression.

LeftHandSideExpression
----------------------
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
MemberExpression :
    MemberBaseExpression MemberAccessChain
```

1. Let ref be ? MemberBaseExpression.Evaluation().
1. Let val be ? GetValue(ref).
1. Return ? MemberAccessChain.ChainEvaluation(ref, val)

```
MemberExpression :
    new MemberBaseExpression Arguments
```
1. Let ref be ? MemberBaseExpression.Evaluation().
1. Let constructor be ? GetValue(ref).
1. Let argList be ? Arguments.ArgumentListEvaluation()
1. If IsConstructor(val) is false, throw a TypeError exception.
1. Return ? Construct(constructor, argList)
```
MemberExpression :
    new MemberBaseExpression MemberAccessChain Arguments
```
1. Let innerRef be ? MemberBaseExpression.Evaluation().
1. Let innerVal be ? GetValue(innerRef).
1. Let ref be ? MemberAccessChain.ChainEvaluation(innerRef, innerVal)
1. Let constructor be ? GetValue(ref).
1. Let argList be ? Arguments.ArgumentListEvaluation()
1. If IsConstructor(val) is false, throw a TypeError exception.
1. Return ? Construct(val, argList).

```
NewExpression :
    new NewExpression
```
1. Let ref be ? NewExpression.Evaluation().
1. Let constructor be ? GetValue(ref).
1. Let argList be a new empty List.
1. If IsConstructor(val) is false, throw a TypeError exception.
1. Return ? Construct(val, argList).


```
CallExpression :
    CallBaseExpression CallAccessChain
```
1. Let ref be ? CallBaseExpression.Evaluation().
1. Let val be ? GetValue(ref).
1. Return ? CallAccessChain.ChainEvaluation(ref, val)

```
OptionalChainingExpression :
    MemberExpression OptionalAccessChain
    CallExpression OptionalAccessChain
    OptionalChainingExpression OptionalAccessChain
``` 
1. Let baseExpression be the first symbol of the rhs of this production (i.e., MemberExpression, CallExpression, or OptionalChainingExpression).
1. Let ref be ? baseExpression.Evaluation().
1. Let val be ? GetValue(ref).
1. If Type(val) is Null or Undefined, then
    1. Return undefined.
1. Return ? OptionalChainExpression.ChainEvaluation(ref, val).


ChainEvaluation(ref, val)
-----------------------
```
MemberAccessChain :
    [ Expression ]
CallAccessChain :
    [ Expression ]
OptionalChainingAccessChain :
    OptionalChainingOperator [ Expression ]
```    
See https://tc39.github.io/ecma262/#sec-property-accessors-runtime-semantics-evaluation, omitting the two first steps,
and replacing baseReference and baseValue with ref and val respectively.

```
MemberAccessChain :
    MemberAccessChain [ Expression ]
CallAccessChain :
    CallAccessChain [ Expression ]
OptionalChainingAccessChain :
    OptionalChainingAccessChain OptionalChainingOperator [ Expression ]
```    

1. Let firstNode be the first symbol of the rhs of this production (i.e., MemberAccessChain, CallAccessChain, or OptionalChainingAccessChain).
1. Let ref2 be ? firstNode.ChainEvaluation(ref, val).
1. Let val2 be ? GetValue(ref2).
1. Continue as in https://tc39.github.io/ecma262/#sec-property-accessors-runtime-semantics-evaluation, omitting the two first steps,
and replacing baseReference and baseValue with ref2 and val2 respectively.

etc.
