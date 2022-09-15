# Extractors for ECMAScript

A proposal to introduce Extractors (a.k.a. "Extractor Objects") to ECMAScript.

Extractors would augment the syntax for _BindingPattern_ and _AssignmentPattern_ to allow for new destructuring forms, as in
the following example:

```js
// binding patterns
const Foo(y) = x;           // instance-array destructuring
const Foo{y} = x;           // instance-object destructuring
const [Foo(y)] = x;         // nesting
const [Foo{y}] = x;         // ..
const { z: Foo(y) } = x;    // ..
const { z: Foo{y} } = x;    // ..
const Foo(Bar(y)) = x;      // ..
const X.Foo(y) = x;         // qualified names (i.e., a.b.c)

// assignment patterns
Foo(y) = x;                 // instance-array destructuring
Foo{y} = x;                 // instance-object destructuring
[Foo(y)] = x;               // nesting
[Foo{y}] = x;               // ..
({ z: Foo(y) } = x);        // ..
({ z: Foo{y} } = x);        // ..
Foo(Bar(y)) = x;            // ..
X.Foo(y) = x;               // qualified names (i.e., a.b.c)
```

In addition, this would leverage the new `Symbol.matcher` built-in symbol added by the Pattern Matching proposal. When
destructuring using the new form, the `Symbol.matcher` method would be called and its result would be destructured instead.

## Status

**Stage:** 1  \
**Champion:** Ron Buckton (@rbuckton)  

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

- Ron Buckton (@rbuckton)

# Motivations

ECMAScript currently has no mechanism for executing user-defined logic during destructuring, which means that operations
related to data validation and transformation may require multiple statements:

```js
function toInstant(value) {
  if (value instanceof Temporal.Instant) {
    return value;
  } else if (value instanceof Date) {
    return Temporal.Instant.fromEpochMilliseconds(+value);
  } else if (typeof value === "string") {
    return Temporal.Instant.from(value);
  } else {
    throw new TypeError();
  }
}

class Book {
  constructor({
    isbn,
    title,
    createdAt = Temporal.Now.instant(),
    modifiedAt = createdAt
  }) {
    this.isbn = isbn;
    this.title = title;
    this.createdAt = toInstant(createdAt);
    // some effort duplicated if `modifiedAt` was `undefined`
    this.modifiedAt = toInstant(modifiedAt);
  }
}

new Book({ isbn: "...", title: "...", createdAt: Temporal.Instant.from("...") });
new Book({ isbn: "...", title: "...", createdAt: new Date() });
new Book({ isbn: "...", title: "...", createdAt: "..." });
```

With _Extractors_, such validation and transformation logic can be encapsulated and reused _inside_ of the binding pattern:

```js
const InstantExtractor = {
  [Symbol.matcher]: value =>
    value instanceof Temporal.Instant ? { matched: true, value: [value] } :
    value instanceof Date ? { matched: true, value: [Temporal.Instant.fromEpochMilliseconds(value.getTime())] } :
    typeof value === "string" ? { matched: true, value: [Temporal.Instant.from(value)] } :
    { matched: false };
  }
};

class Book {
  constructor({
    isbn,
    title,
    // Extract `createdAt` as an Instant
    InstantExtractor(createdAt) = Temporal.Now.instant(),
    InstantExtractor(modifiedAt) = createdAt
  }) {
    this.isbn = isbn;
    this.title = title;
    this.createdAt = createdAt;
    this.modifiedAt = modifiedAt;
  }
}

new Book({ isbn: "...", title: "...", createdAt: Temporal.Instant.from("...") });
new Book({ isbn: "...", title: "...", createdAt: new Date() });
new Book({ isbn: "...", title: "...", createdAt: "..." });
```

This would also be extremely useful when paired with a forthcoming `enum` proposal with support for Algebraic Data Types (ADT):

```js
// Rust-like enum of algebraic data types:
enum Option of ADT {
  Some(value),
  None
}

// construction
const x = Option.Some(1);

// destructuring
const Option.Some(y) = x;
y; // 1

// pattern matching
match (x) {
  when Option.Some(y): console.log(y); // 1
  when Option.None: console.log("none");
}
```

```js
// Another ADT enum example:
enum Message of ADT {
  Quit,
  Move{x, y},
  Write(message),
  ChangeColor(r, g, b),
}

// construction
const msg1 = Message.Move{ x: 10, y: 10 }; // NOTE: possible novel sytax for enum construction
const msg2 = Message.Write("Hello");
const msg3 = Message.ChangeColor(0x00, 0xff, 0xff);

// destructuring
const Message.Move{ x, y } = msg1;      // x: 10, y: 10
const Message.Write(message) = msg2;    // message: "Hello"
const Message.ChangeColor(r, g, b);     // r: 0, g: 255, b: 255

// pattern matching
match (msg) {
  when Message.Move{ x, y }: ...;
  when Message.Write(message): ...;
  when Message.ChangeColor(r, g, b): ...;
  when Message.Quit: ...;
}
```

# Proposed Solution

_Extractors_ are loosely based on Scala's [Extractor Objects](https://docs.scala-lang.org/tour/extractor-objects.html) and
Rust's [Pattern Matching](https://doc.rust-lang.org/book/ch18-00-patterns.html). Extractors extend the syntax for
_BindingPattern_ and _AssignmentPattern_ to allow for the evaluation of user-defined logic for validation and transformation.

Extractors come in two flavors: _Array Extractors_, which perform array destructuring on a successful match result, and
_Object Extractors_, which perform object destructuring. Both types of extractors start with a reference to a value in scope
using a _QualifiedName_, which is essentially an _IdentifierReference_ (i.e., `Point`, `InstantExtractor`, etc.) or a 
dotted-name (i.e., `Option.Some`, `Message.Move`, etc.).

When an Extractor is evaluated during destructuring, its _QualifiedName_ is evaluated, and that evaluated result's `[Symbol.matcher]` 
method is invoked with the current value to be destructured, returning a _Match Result_ object like `{ matched: boolean, value: object }`.

If `matched` is `true`, the `value` property is further destructured based on the type of Extractor that was defined. If `matched` is 
`false`, a *TypeError* is thrown.

## Array Extractors

An _Array Extractor_ consists of a _QualifiedName_ followed by a parenthesized list of additional destructuring patterns:

```js
// binding pattern
let Foo(a, { b }, [c]) = ...;

// assignment pattern
Foo(a, { b }, [c]) = ...;
```

Parentheses (`()`) are used instead of square brackets (`[]`) for several reasons:

- Avoids collisions with a _ElementAccessExpression_ when the extractor is part of an assignment pattern:
  ```js
  Option.Some[value] = opt; // already an element access expression
  ```
- Ensures a consistent syntax between binding and assignment patterns:
  ```js
  let Option.Some(value) = opt;
  Option.Some(value) = opt;
  ```
- Allows destructuring (and pattern matching) to mirror construction/application:
  ```js
  let opt = Option.Some(x);
  let Option.Some(y) = opt;

  opt = Option.Some(x);
  Option.Some(y) = opt;
  ```

## Object Extractors

An _Object Extractor_ consists of a _QualifiedName_ followed by an object destructuring pattern:

```js
// binding pattern
const Message.Move{ x, y } = ...;

// assignment pattern
(Message.Move{ x, y } = ...);
```

Curly braces (`{}`) are used here to be otherwise consistent with normal object destructuring. Outermost parentheses would also
likely be required to avoid too large of a syntax carve-out for `identifier { ... }` at the statement level.

The use of curly braces here is also intended to mirror a potential future property-literal construction syntax that might be used
by Algebraic Data Types or other constructors, i.e.:

```js
const enum Message of ADT {
  Move{ x, y },
  Write(text),
  Quit
}

// ADT construction
const msg = Message.Move{ x: 10, y: 20 };

// property-literal construction potentially used by fixed-shape objects (i.e., "struct"),
// or a user-defined construction mechanism similar to tagged templates or via a built-in-symbol named method:
struct Point { x, y };
const pt = Point{ x: 10, y: 20 };
```

# Prior Art

- [Scala Extractor Objects](https://docs.scala-lang.org/tour/extractor-objects.html)
- [C# 8.0 Deconstruct](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/deconstruct)
- [F# Active Patterns](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/active-patterns)
- [Rust Enums](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html)
- [Rust Pattern Matching](https://doc.rust-lang.org/book/ch18-00-patterns.html)

# Related Proposals

- [Collection Literals](https://github.com/zkat/proposal-collection-literals) (Withdrawn)
- [Pattern Matching](https://github.com/tc39/proposal-pattern-matching) (Stage 1)
- [Enums and Algebraic Data Types](https://github.com/Jack-Works/proposal-enum) (Stage 0, not yet accepted)

# Examples

The examples in this section use a desugaring to explain the underlying semantics, given the following helper:

```js
function %InvokeCustomMatcher%(val, matchable) {
    // see https://tc39.es/proposal-pattern-matching/#sec-custom-matcher
}

function %InvokeCustomMatcherOrThrow%(val, matchable) {
  const result = %InvokeCustomMatcher%(val, matchable);
  if (result === ~not-matched~) {
    throw new TypeError();
  }
  return result;
}
```

## Extractor Destructuring

The statement

```js
const Foo(y) = x;
```

is approximately the same as the following transposed representation

```js
const [y] = %InvokeCustomMatcherOrThrow%(Foo, x);
```

The statement

```js
const Foo{y} = x;
```

is approximately the same as the following transposed representation

```js
const {y} = %InvokeCustomMatcherOrThrow%(Foo, x);
```

## Nested Extractor Destructuring

The statement

```js
const Foo(Bar(y)) = x;
```

is approximately the same as the following transposed representation

```js
const [_a] = %InvokeCustomMatcherOrThrow%(Foo, x);
const [y] = %InvokeCustomMatcherOrThrow%(Bar, _a);
```

## Custom Logic During Destructuring

Given the following definition

```js
const MapExtractor = {
  [Symbol.matcher](map) {
    const obj = {};
    for (const [key, value] of map) {
      obj[typeof key === "symbol" ? key : `${key}`] = value;
    }
    return { matched: true, value: obj };
  }
};

const obj = {
    map: new Map([["a", 1], ["b", 2]])
};
```

The statement

```js
const { map: MapExtractor{ a, b } } = obj;
```

is approximately the same as the following transposed representation

```js
const { map: _temp } = obj;
const { a, b } = %InvokeCustomMatcherOrThrow%(MapExtractor, _temp);
```

## Regular Expressions

```js
// potentially built-in as part of Pattern Matching
RegExp.prototype[Symbol.matcher] = function (value) {
  const match = this.exec(value);
  if (match === null) return { matched: false };

  return {
    matched: true,
    value: {
      // spread in named capture groups for use with object destructuring
      ...match.groups,

      // iterator for use with array destructuring
      [Symbol.iterator]: () => match[Symbol.iterator]()
    }
  }
};

const IsoDate = /^(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})$/;
const IsoTime = /^(?<hours>\d{2}):(?<minutes>\d{2}):(?<seconds>\d{2})$/;
const IsoDateTime = /^(?<date>[^TZ]+)T(?<time>[^Z]+)Z/;

// match `input`, extract, and destructure (or throw if match fails) using...

// ...an object extractor
const IsoDate{ year, month, day } = input;

// ...an array extractor
const IsoDate(, year, month, day) = input;

// concise, multi-step extraction using nested destructuring:
const IsoDateTime{
    date: IsoDate{ year, month, day },
    time: IsoTime{ hours, minutes, seconds }
} = input;
// 1. Matches `input` via `IsoDatTime` RegExp and extracts `date` and `time`
// 2. Matches `date` via `IsoDate` RegExp and extracts `year`, `month`, and `day` as lexical bindings
// 3. Matches `time` vai `IsoTime` RegExp and extracts `hours`, `minutes`, and `seconds` as lexical bindings
```

# Potential Grammar

The grammar definition in this section is very early and subject to change.

```diff grammarkdown
++  QualifiedName[Yield, Await] :
++      IdentifierReference[?Yield, ?Await]
++      QualifiedName[?Yield, ?Await] `.` IdentifierName

    BindingPattern[Yield, Await] :
        ObjectBindingPattern[?Yield, ?Await]
        ArrayBindingPattern[?Yield, ?Await]
++      QualifiedName[?Yield, ?Await] ExtractorBindingPattern[?Yield, ?Await]

++  ExtractorBindingPattern[Yield, Await] :
++      ExtractorObjectBindingPattern[?Yield, ?Await]
++      ExtractorArrayBindingPattern[?Yield, ?Await]

++  ExtractorObjectBindingPattern[Yield, Await] :
++      ObjectBindingPattern[?Yield, ?Await]

++  ExtractorArrayBindingPattern[Yield, Await] :
++      `(` Elision? BindingRestElement[?Yield, ?Await]? `)`
++      `(` BindingElementList[?Yield, ?Await] `)`
++      `(` BindingElementList[?Yield, ?Await] `,` Elision? BindingRestElement[?Yield, ?Await]? `)`

    AssignmentPattern[Yield, Await] :
        ObjectAssignmentPattern[?Yield, ?Await]
        ArrayAssignmentPattern[?Yield, ?Await]
++      QualifiedName[?Yield, ?Await] ExtractorAssignmentPattern[?Yield, ?Await]

++  ExtractorAssignmentPattern[Yield, Await] :
++      ExtractorObjectAssignmentPattern[?Yield, ?Await]
++      ExtractorArrayAssignmentPattern[?Yield, ?Await]

++  ExtractorObjectAssignmentPattern[Yield, Await] :
++      ObjectAssignmentPattern[?Yield, ?Await]

++  ExtractorArrayAssignmentPattern[Yield, Await] :
++      `(` Elision? BindingRestElement[?Yield, ?Await]? `)`
++      `(` BindingElementList[?Yield, ?Await] `)`
++      `(` BindingElementList[?Yield, ?Await] `,` Elision? BindingRestElement[?Yield, ?Await]? `)`

++  FunctionCall[Yield, Await] :
++      CallExpression[?Yield, ?Await] Arguments[?Yield, ?Await]

    CallExpression[Yield, Await] :
        CoverCallExpressionAndAsyncArrowHead[?Yield, ?Await]
        SuperCall[?Yield, ?Await]
        ImportCall[?Yield, ?Await]
++      FunctionCall[?Yield, ?Await]
--      CallExpression[?Yield, ?Await] Arguments[?Yield, ?Await]
        CallExpression[?Yield, ?Await] `[` Expression[+In, ?Yield, ?Await] `]`
        CallExpression[?Yield, ?Await] `.` IdentifierName
        CallExpression[?Yield, ?Await] TemplateLiteral[?Yield, ?Await, +Tagged]
        CallExpression[?Yield, ?Await] `.` PrivateIdentifier
```

# Potential Semantics

The semantics in this section are very early and subject to change.

## 8.5.2 &mdash; Runtime Semantics: BindingInitialization

The syntax-directed operation BindingInitialization takes arguments _value_ and _environment_.

```diff grammarkdown
++  BindingPattern : QualifiedName ExtractorBindingPattern
```

1. <ins>Let _ref_ be the result of evaluating _QualifiedName_.</ins>
1. <ins>Let _extractor_ be ? GetValue(_ref_).</ins>
1. <ins>Let _obj_ be ? InvokeCustomMatcherOrThrow(_extractor_, _value_).</ins>
1. <ins>Return the result of performing BindingInitialization of _ExtractorBindingPattern_ with arguments _obj_ and _environment_.</ins>

```diff grammarkdown
++  ExtractorBindingPattern : ExtractorArrayBindingPattern
```

1. <ins>Let _iteratorRecord_ be ? GetIterator(_value_).</ins>
1. <ins>Let _result_ be IteratorBindingInitialization of _ExtractorBindingPattern_ with arguments _iteratorRecord_ and _environment_.</ins>
1. <ins>If _iteratorRecord_.\[\[Done]] is **false**, return ? IteratorClose(_iteratorRecord_, _result_).</ins>
1. <ins>Return _result_.</ins>

## 8.5.2.2 &mdash; <ins>Runtime Semantics: InvokeCustomMatcherOrThrow ( _val_, _matchable_ )</ins>

1. <ins>Let _result_ be ? InvokeCustomMatcher(_val_, _matchable_).</ins>
1. <ins>If _result_ is ~not-matched~, throw a **TypeError** exception.</ins>
1. <ins>Return _result_.</ins>

## 8.5.3 &mdash; Runtime Semantics: IteratorBindingInitialization

The syntax-directed operation IteratorBindingInitialization takes arguments _iteratorRecord_ and _environment_.

```diff grammarkdown
    ArrayBindingPattern : `[` `]`
++  ExtractorBindingPattern : `(` `)`
```

1. Return NormalCompletion(empty).

```diff grammarkdown
    ArrayBindingPattern : `[` Elision `]`
++  ExtractorBindingPattern : `(` Elision `)`
```

1. Return the result of performing IteratorDestructuringAssignmentEvaluation of _Elision_ with _iteratorRecord_ as the argument.

```diff grammarkdown
    ArrayBindingPattern : `[` Elision? BindingRestElement `]`
++  ExtractorBindingPattern : `(` Elision? BindingRestElement `)`
```

1. If _Elision_ is present, then
    1. Perform ? IteratorDestructuringAssignmentEvaluation of _Elision_ with _iteratorRecord_ as the argument.
1. Return the result of performing IteratorBindingInitialization for _BindingRestElement_ with _iteratorRecord_ and _environment_ as arguments.

```diff grammarkdown
    ArrayBindingPattern : `[` BindingElementList `,` Elision `]`
++  ExtractorBindingPattern : `(` BindingElementList `,` Elision `)`
```

1. Perform ? IteratorBindingInitialization for _BindingElementList_ with _iteratorRecord_ and _environment_ as arguments.
1. Return the result of performing IteratorDestructuringAssignmentEvaluation of _Elision_ with _iteratorRecord_ as the argument.

```diff grammarkdown
    ArrayBindingPattern : `[` BindingElementList `,` Elision? BindingRestElement `]`
++  ExtractorBindingPattern : `(` BindingElementList `,` Elision? BindingRestElement `)`
```

1. Perform ? IteratorBindingInitialization for _BindingElementList_ with _iteratorRecord_ and _environment_ as arguments.
1. If _Elision_ is present, then
    1. Perform ? IteratorDestructuringAssignmentEvaluation of _Elision_ with _iteratorRecord_ as the argument.
1. Return the result of performing IteratorBindingInitialization for _BindingRestElement_ with _iteratorRecord_ and _environment_ as arguments.

## 13.15.1 &mdash; Static Semantics: Early Errors

```diff grammarkdown
    AssignmentExpression : LeftHandSideExpression = AssignmentExpression
```

If _LeftHandSideExpression_ is an _ObjectLiteral_<del> or</del><ins>,</ins> an _ArrayLiteral_<ins>, or a _FunctionCall_</ins> the following Early Error rules are applied:

- It is a Syntax Error if _LeftHandSideExpression_ is not covering an _AssignmentPattern_.
- All Early Error rules for _AssignmentPattern_ and its derived productions also apply to the _AssignmentPattern_ that is covered
  by _LeftHandSideExpression_.

If _LeftHandSideExpression_ is neither an _ObjectLiteral_<ins>,</ins> nor an _ArrayLiteral_<ins>, nor a _FunctionCall_</ins>, the following Early Error rule is applied:

- It is a Syntax Error if AssignmentTargetType of _LeftHandSideExpression_ is not `simple`.

```diff grammarkdown
    AssignmentExpression :
        LeftHandSideExpression AssignmentOperator AssignmentExpression
        LeftHandSideExpression `&&=` AssignmentExpression
        LeftHandSideExpression `||=` AssignmentExpression
        LeftHandSideExpression `??=` AssignmentExpression
```

- It is a Syntax Error if AssignmentTargetType of _LeftHandSideExpression_ is not `simple`.

## 13.15.2 &mdash; Runtime Semantics: Evaluation

```diff grammarkdown
    AssignmentExpression : LeftHandSideExpression `=` AssignmentExpression
```

1. If _LeftHandSideExpression_ is neither an _ObjectLiteral_<ins>,</ins> nor an ArrayLiteral<ins>, nor a _FunctionCall_</ins>, then
    1. Let _lref_ be the result of evaluating _LeftHandSideExpression_.
    1. ReturnIfAbrupt(_lref_).
    1. If IsAnonymousFunctionDefinition(_AssignmentExpression_) and IsIdentifierRef of _LeftHandSideExpression_ are both **true**, then
        1. Let _rval_ be NamedEvaluation of _AssignmentExpression_ with argument _lref_.\[\[ReferencedName]].
    1. Else,
        1. Let _rref_ be the result of evaluating _AssignmentExpression_.
        1. Let _rval_ be ? GetValue(_rref_).
    1. Perform ? PutValue(lref, _rval_).
    1. Return _rval_.
2. Let _assignmentPattern_ be the _AssignmentPattern_ that is covered by _LeftHandSideExpression_.
3. Let _rref_ be the result of evaluating _AssignmentExpression_.
4. Let _rval_ be ? GetValue(_rref_).
5. Perform ? DestructuringAssignmentEvaluation of _assignmentPattern_ using _rval_ as the argument.
6. Return _rval_.

## 13.15.5.2 &mdash; Runtime Semantics: DestructuringAssignmentEvaluation

The syntax-directed operation DestructuringAssignmentEvaluation takes argument _value_. It is defined piecewise over the following productions:

```diff grammarkdown
++  AssignmentPattern : QualifiedName ExtractorAssignmentPattern
```

1. <ins>Let _ref_ be the result of evaluating _QualifiedName_.</ins>
1. <ins>Let _extractor_ be ? GetValue(_ref_).</ins>
1. <ins>Let _obj_ be ? InvokeCustomMatcherOrThrow(_extractor_, _value_).</ins>
1. <ins>Return the result of performing DestructuringAssignmentEvaluation of _ExtractorAssignmentPattern_ with argument _obj_.</ins>

```diff grammarkdown
    ArrayAssignmentPattern : `[` `]`
++  ExtractorArrayAssignmentPattern : `(` `)`
```

1. Let _iteratorRecord_ be ? GetIterator(_value_).
1. Return ? IteratorClose(_iteratorRecord_, NormalCompletion(empty)).

```diff grammarkdown
    ArrayAssignmentPattern : `[` Elision `]`
++  ExtractorArrayAssignmentPattern : `(` Elision `)`
```


1. Let _iteratorRecord_ be ? GetIterator(value).
1. Let _result_ be IteratorDestructuringAssignmentEvaluation of _Elision_ with argument _iteratorRecord_.
1. If _iteratorRecord_.[[Done]] is false, return ? IteratorClose(_iteratorRecord_, _result_).
1. Return _result_.

```diff grammarkdown
    ArrayAssignmentPattern : `[` Elision? AssignmentRestElement `]`
++  ExtractorArrayAssignmentPattern : `(` Elision? AssignmentRestElement `)`
```

1. Let _iteratorRecord_ be ? GetIterator(_value_).
1. If _Elision_ is present, then
    1. Let _status_ be IteratorDestructuringAssignmentEvaluation of _Elision_ with argument _iteratorRecord_.
    1. If _status_ is an abrupt completion, then
        1. Assert: _iteratorRecord_.[[Done]] is **true**.
        1. Return Completion(_status_).
1. Let _result_ be IteratorDestructuringAssignmentEvaluation of _AssignmentRestElement_ with argument _iteratorRecord_.
1. If _iteratorRecord_.[[Done]] is **false**, return ? IteratorClose(_iteratorRecord_, result).
1. Return result.

```diff grammarkdown
    ArrayAssignmentPattern : `[` AssignmentElementList `]`
++  ExtractorArrayAssignmentPattern : `(` AssignmentElementList `)`
```

1. Let _iteratorRecord_ be ? GetIterator(_value_).
1. Let _result_ be IteratorDestructuringAssignmentEvaluation of _AssignmentElementList_ with argument _iteratorRecord_.
1. If _iteratorRecord_.[[Done]] is **false**, return ? IteratorClose(_iteratorRecord_, _result_).
1. Return _result_.

```diff grammarkdown
    ArrayAssignmentPattern : `[` AssignmentElementList `,` Elision? AssignmentRestElement? `]`
++  ExtractorArrayAssignmentPattern : `(` AssignmentElementList `,` Elision? AssignmentRestElement? `)`
```

1. Let _iteratorRecord_ be ? GetIterator(_value_).
1. Let _status_ be IteratorDestructuringAssignmentEvaluation of _AssignmentElementList_ with argument _iteratorRecord_.
1. If _status_ is an abrupt completion, then
    1. If _iteratorRecord_.[[Done]] is **false**, return ? IteratorClose(_iteratorRecord_, _status_).
    1. Return Completion(_status_).
1. If _Elision_ is present, then
    1. Set _status_ to the result of performing IteratorDestructuringAssignmentEvaluation of _Elision_ with _iteratorRecord_ as the argument.
    1. If _status_ is an abrupt completion, then
        1. Assert: _iteratorRecord_.[[Done]] is **true**.
        1. Return Completion(_status_).
1. If _AssignmentRestElement_ is present, then
    1. Set _status_ to the result of performing IteratorDestructuringAssignmentEvaluation of _AssignmentRestElement_ with _iteratorRecord_ as the
       argument.
1. If _iteratorRecord_.[[Done]] is **false**, return ? IteratorClose(_iteratorRecord_, _status_).
1. Return Completion(_status_).

# API

This proposal would adopt (and continue to align with) the behavior of _Custom Matchers_ from the Pattern Matching proposal:

- A _Custom Matcher_ is a regular ECMAScript Object value with a `[Symbol.matcher]` method that accepts a single argument and
  returns a _Match Result_.
- A _Match Result_ is a regular ECMAScript Object value with a `matched` property whose value is a Boolean, and a `value` property
  whose value will be destructured by the relevant Extractor pattern.
- `Symbol.matcher` is a built-in Symbol value.

# Relation to Pattern Matching

We believe that Extractors would also be extremely valuable as part of the Pattern Matching Proposal, and intend to discuss adoption with
the champions should this proposal be adopted.

Extractors could easily be added to _MatchPattern_ using the same syntax as proposed for destructuring, which would allow for more concise
and potentially more readily understood code:

```js
match (opt) {
  // without extractors
  when (${Option.Some} with [value]): ...;

  // with extractors
  when (Option.Some(value)): ...;
}

match (msg) {
  // without extractors
  when (${Message.Move} with { x, y }): ...;

  // with extractors
  when (Message.Move{ x, y }): ...;
}
```

This is even more evident with respect to complex, nested patterns:

```js
match (opt) {
  // without extractors
  when (${Option.Some} with [${Message.Move} with { x, y }]): ...;
  when (${Option.Some} with [${Message.Write} with [text]]): ...;

  // with extractors
  when (Option.Some(Message.Move{ x, y })): ...;
  when (Option.Some(Message.Write(text))): ...;
}
```

# Relation to Enums and Algebraic Data Types

We strongly believe that ECMAScript will eventually adopt some form of the current `enum` proposal, given the particular value
that Algebraic Data Types could provide. The Enum proposal would strongly favor consistent and coherent syntax between 
_declaration_, _construction_, _destructuring_, and _pattern matching_, as in the following example:

```js
enum Message of ADT {
  Quit,
  Move{x, y},
  Write(message),
  ChangeColor(r, g, b),
}

// construction
const msg1 = Message.Move{ x: 10, y: 10 };
const msg2 = Message.Write("Hello");
const msg3 = Message.ChangeColor(0x00, 0xff, 0xff);

// destructuring
const Message.Move{x, y} = msg1;        // x: 10, y: 10
const Message.Write(message) = msg2;    // message: "Hello"
const Message.ChangeColor(r, g, b);     // r: 0, g: 255, b: 255

// pattern matching
match (msg) {
  when Message.Move{x, y}: ...;
  when Message.Write(message): ...;
  when Message.ChangeColor(r, g, b): ...;
  when Message.Quit: ...;
}
```

Here, declaration, construction, destructuring, and pattern matching are consistent for ADT enum members and values:

```js
enum Message of ADT { Move{ x, y } }      // declaration
const msg =   Message.Move{ x, y };       // construction
const         Message.Move{ x, y } = msg; // destructuring
match (msg) {
  when        Message.Move{ x, y }: ...;  // pattern matching
}
```

```js
enum Message of ADT { Write(message) }      // declaration
const msg =   Message.Write(message);       // construction
const         Message.Write(message) = msg; // destructuring
match (msg) {
  when        Message.Write(message): ...;  // pattern matching
}
```


# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [x] Illustrative [examples][Examples] of usage.
* [x] High-level [API][API].

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].
* [ ] [Transpiler support][Transpiler] (_Optional_).

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].



<!-- # References -->

<!-- Links to other specifications, etc. -->


<!-- * [Title](url) -->


<!-- # Prior Discussion -->

<!-- Links to prior discussion topics on https://esdiscuss.org -->


<!-- * [Subject](https://esdiscuss.org) -->


<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: #todo
[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
