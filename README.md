# Extractors for ECMAScript

A proposal to introduce Extractors (a.k.a. "Extractor Objects") to ECMAScript.

Extractors would augment the syntax for _BindingPattern_ and _AssignmentPattern_ to allow for new destructuring forms,
as in the following example:

```js
// binding patterns
const Foo(y) = x;           // instance-array destructuring
const Foo([y]) = x;         // nested array destructuring
const Foo({y}) = x;         // nested object destructuring
const [Foo(y)] = x;         // nesting
const { z: Foo(y) } = x;    // ..
const Foo(Bar(y)) = x;      // ..
const X.Foo(y) = x;         // qualified names (i.e., a.b.c)

// assignment patterns
Foo(y) = x;                 // instance-array destructuring
Foo([y]) = x;               // nestedarray destructuring
Foo({y}) = x;               // nested object destructuring
[Foo(y)] = x;               // nesting
({ z: Foo(y) } = x);        // ..
Foo(Bar(y)) = x;            // ..
X.Foo(y) = x;               // qualified names (i.e., a.b.c)
```

In addition, this would leverage the new `Symbol.customMatcher` built-in symbol added by the Pattern Matching proposal. When
destructuring using the new form, the `Symbol.customMatcher` method would be called and its result would be destructured instead.

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
  [Symbol.customMatcher](value) {
    if (value instanceof Temporal.Instant) {
      return [value];
    } else if (value instanceof Date) {
      return [Temporal.Instant.fromEpochMilliseconds(+value)];
    } else if (typeof value === "string") {
      return [Temporal.Instant.from(value)];
    }
  }
};

class Book {
  constructor({
    isbn,
    title,
    // Extract `createdAt` as an Instant
    createdAt: InstantExtractor(createdAt) = Temporal.Now.instant(),
    modifiedAt: InstantExtractor(modifiedAt) = createdAt
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
  Move({x, y}),
  Write(message),
  ChangeColor(r, g, b),
}

// construction
const msg1 = Message.Move({ x: 10, y: 10 });
const msg2 = Message.Write("Hello");
const msg3 = Message.ChangeColor(0x00, 0xff, 0xff);

// destructuring
const Message.Move({ x, y }) = msg1;    // x: 10, y: 10
const Message.Write(message) = msg2;    // message: "Hello"
const Message.ChangeColor(r, g, b) = msg3;     // r: 0, g: 255, b: 255

// pattern matching
match (msg) {
  when Message.Move({ x, y }): ...;
  when Message.Write(message): ...;
  when Message.ChangeColor(r, g, b): ...;
  when Message.Quit: ...;
}
```

# Proposed Solution

_Extractors_ are loosely based on Scala's [Extractor Objects](https://docs.scala-lang.org/tour/extractor-objects.html)
and Rust's [Pattern Matching](https://doc.rust-lang.org/book/ch18-00-patterns.html). Extractors extend the syntax for
_BindingPattern_ and _AssignmentPattern_ to allow for the evaluation of user-defined logic for validation and
transformation.

Extractors perform array destructuring on a successful match result, starting with a reference to a value in scope
using an _ExtractorMemberExpression_, which is essentially an _IdentifierReference_ (i.e., `Point`, `InstantExtractor`,
etc.) or a dotted-name (i.e., `Option.Some`, `Message.Move`, etc.).

When an Extractor is evaluated during destructuring, its _ExtractorMemberExpression_ is evaluated, and that evaluated
result's `[Symbol.customMatcher]()` method is invoked with the current value to be destructured, returning an iterable
object that indicates the match succeeded, and the extracted elements to use for further destructuring. For the purpose
of destructuring, any other value will produce a *TypeError*. In the case of pattern matching, `true` and `false` are
also valid return values.

An _Extractor_ consists of an _ExtractorMemberExpression_ followed by a parenthesized list of additional destructuring patterns:

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

## Object Destructuring using Extractors

Extractors do not introduce a novel syntax for object extraction. Instead, an Extractor can return a single element
match result containing the object to be further destructured:

```js
// binding pattern
const Message.Move({ x, y }) = ...;

// assignment pattern
(Message.Move({ x, y }) = ...);
```

### Iterable Wrapper Overhead

It's important to note that this has the overhead of allocating an iterable wrapper object to perform further
destructuring. If necessary, this overhead could be removed if we consider an alternative representation for a match
result that indicates result is the sole extracted value, i.e.:

```js
const Message = {
  Move: class Move {
    #x;
    #y;
    ...
    static [Symbol.match](value) {
      if (value instanceof Message.Move) {
        // 'match: "unary"' indicates that 'value' is the sole extracted value
        return { match: "unary", value: { x: value.#x, y: value.#y } };
      }
      return false;
    }
  }
};

const Message.Move({ x, y }) = ...;
```

### Future Object Extractor Syntax

It is possible that a future property-literal construction syntax that might be used by Algebraic Data Types or other
constructors, i.e.:

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

However, such a syntax is currently out of scope for this proposal. In addition, introducing an `Identifier {` syntax
for extractors has the high likelihood of carving off too much syntax space that could be used by other proposals. As
a result, an "Object Extractor" like syntax like `const Point{ x, y } = p` is not being considered part of this proposal
at this time.

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

<a name="extractor-destructuring"></a>
The examples in this section use a desugaring to explain the underlying semantics, given the following helper:

```js
function %InvokeCustomMatcherOrThrow%(extractor, subject, receiver) {
  if (typeof extractor !== "object" || extractor === null) {
    throw new TypeError();
  }
  const f = extractor[Symbol.customMatcher];
  if (typeof f !== "function") {
    throw new TypeError();
  }
  const result = f.apply(extractor, [subject, "list", receiver]);
  if (typeof result !== "object" || result === null) {
    throw new TypeError();
  }
  return result;
}
```

## `const C(x) = subject`

Given,

```js
class C {
  #data;
  constructor(data) {
    this.#data = data;
  }
  [Symbol.customMatcher](subject) {
    return #data in subject && [this.#data];
  }
}

const subject = new C("data");

const C(x) = subject;
x; // "data"
```

The statement

```js
const C(x) = subject;
```

is approximately the same as the transposed representation

```js
const [x] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
```

such that `x` results in the value `"data"`.

## `const C(x, y) = subject`

Given,

```js
class C {
  #first;
  #second;
  constructor(first, second) {
    this.#first = first;
    this.#second = second;
  }
  [Symbol.customMatcher](subject) {
    return #first in subject && [this.#first, this.#second];
  }
}

const subject = new C(1, 2);

const C(x, y) = subject;
x; // 1
y; // 2
```

The statement

```js
const C(x, y) = subject;
```

is approximately the same as the transposed representation

```js
const [x, y] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
```

such that `x` and `y` result in the values `1` and `2`, respectively.

## `const C(x, ...y) = subject`

Given,

```js
class C {
  #first;
  #second;
  #third;
  constructor(first, second, third) {
    this.#first = first;
    this.#second = second;
    this.#third = third;
  }
  [Symbol.customMatcher](subject) {
    return #first in subject && [this.#first, this.#second, this.#third];
  }
}

const subject = new C(1, 2, 3);

const C(x, ...y) = subject;
x; // 1
y; // [2, 3]
```

The statement

```js
const C(x, ...y) = subject;
```

is approximately the same as the transposed representation

```js
const [x, ...y] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
```

such that `x` and `y` result in the values `1` and `[2, 3]`, respectively.


## `const C(x = -1, y) = subject`

Given,

```js
class C {
  #first;
  #second;
  constructor(first, second) {
    this.#first = first;
    this.#second = second;
  }
  [Symbol.customMatcher](subject) {
    return #first in subject && [this.#first, this.#second];
  }
}

const subject = new C(undefined, 2);

const C(x = -1, y) = subject;
x; // -1
y; // 2
```

The statement

```js
const C(x = -1, y) = subject;
```

is approximately the same as the transposed representation

```js
const [x = -1, y] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
```

such that `x` and `y` result in the values `-1` and `2`, respectively.

## `const C({ x }) = subject`

Given,

```js
class C {
  #data;
  constructor(data) {
    this.#data = data;
  }
  [Symbol.customMatcher](subject) {
    return #data in subject && [this.#data];
  }
}

const subject = new C({ x: 1, y: 2 });

const C({ x, y }) = subject;
x; // 1
y; // 2
```

The statement

```js
const C({ x, y }) = subject;
```

is approximately the same as the transposed representation

```js
const [{ x, y }] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
```

such that `x` and `y` have the values `1` and `2`, respectively.

## `const C(D(x)) = subject`

Given,

```js
class C {
  #data1;
  constructor(data1) {
    this.#data1 = data1;
  }
  [Symbol.customMatcher](subject) {
    return #data1 in subject && [this.#data1];
  }
}

class D {
  #data2;
  constructor(data2) {
    this.#data2 = data2;
  }
  [Symbol.customMatcher](subject) {
    return #data2 in subject && [this.#data2];
  }
}

const subject = new C(D("data"));

const C(D(x)) = subject;
x; // "data"
```

The statement

```js
const C(D(x)) = subject;
```

is approximately the same as the transposed representation

```js
const [_a] = %InvokeCustomMatcherOrThrow%(C, subject, undefined);
const [x] = %InvokeCustomMatcherOrThrow%(D, _a, undefined);
```

such that `x` results in the value `"data"`.



## Custom Logic During Destructuring

Given,

```js
const MapExtractor = {
  [Symbol.customMatcher](map) {
    const obj = {};
    for (const [key, value] of map) {
      obj[typeof key === "symbol" ? key : `${key}`] = value;
    }
    return [obj];
  }
};

const obj = {
    map: new Map([["a", 1], ["b", 2]])
};

const { map: MapExtractor({ a, b }) } = obj;
a; // 1
b; // 2
```

The statement

```js
const { map: MapExtractor({ a, b }) } = obj;
```

is approximately the same as the transposed representation

```js
const { map: _temp } = obj;
const [{ a, b }] = %InvokeCustomMatcherOrThrow%(MapExtractor, _temp, undefined);
```

such that `a` and `b` result in the values `1` and `2`, respectively.

## Regular Expressions

```js
// potentially built-in as part of Pattern Matching
RegExp.prototype[Symbol.customMatcher] = function (value) {
  const match = this.exec(value);
  return !!match && [match];
};

const IsoDate = /^(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})$/;
const IsoTime = /^(?<hours>\d{2}):(?<minutes>\d{2}):(?<seconds>\d{2})$/;
const IsoDateTime = /^(?<date>[^TZ]+)T(?<time>[^Z]+)Z/;

// match `input`, extract, and destructure (or throw if match fails) using...

// ...a nested-object extractor
const IsoDate({ groups: { year, month, day } }) = input;

// ...a nested-array extractor
const IsoDate([, year, month, day]) = input;

// concise, multi-step extraction using nested destructuring:
const IsoDateTime({
  groups: {
    date: IsoDate({ groups: { year, month, day } }),
    time: IsoTime({ groups: { hours, minutes, seconds }})
  }
}) = input;
// 1. Matches `input` via `IsoDatTime` RegExp and extracts `date` and `time`
// 2. Matches `date` via `IsoDate` RegExp and extracts `year`, `month`, and `day` as lexical bindings
// 3. Matches `time` vai `IsoTime` RegExp and extracts `hours`, `minutes`, and `seconds` as lexical bindings
```

## Receivers: `const obj.extractor(x) = subject`

When the _ExtractorMemberExpression_ results in a Reference, the receiver is preserved and passed on to the custom
matcher:

Given,

```js
class C {
  #f;
  constructor(f) {
    this.#f = f;
  }
  extractor = {
      [Symbol.customMatcher](subject, _kind, receiver) {
        return receiver.#f(subject);
      }
  };
}

const obj = new C(data => data.toUpperCase());
const subject = "data";

const obj.extractor(x) = subject;
x; // "DATA"
```

The statement

```js
const obj.extractor(x) = subject;
```

is approximately the same as the transposed representation

```js
const _receiver = obj;
const [x] = %InvokeCustomMatcherOrThrow%(_receiver.extractor, subject, _receiver);
```

such that `x` results in the value `"DATA"`.

# Potential Grammar

For the proposed grammar, see [the specification text](https://tc39.es/proposal-extractors).

# Potential Semantics

For the proposed semantics, see [the specification text](https://tc39.es/proposal-extractors).

# API

This proposal would adopt (and continue to align with) the behavior of _Custom Matchers_ from the Pattern Matching proposal:

- A _Custom Matcher_ is a regular ECMAScript Object value with a `[Symbol.customMatcher]` method that accepts a three
  arguments: `subject` (the value to match against), `hint` (either `"boolean"` or `"list"`), and `receiver`,
  and returns either a Boolean or an _Iterable_, depending on the value of `hint`.
- When `hint` is `"boolean"`, the return value will be coerced to a Boolean via the ToBoolean() abstract operation. When
  `hint` is `"list"`, the return value value must either be an _Iterable_ object or a falsy value (i.e., `false`, `0`,
  `null`, `undefined`, etc.).
  - In Pattern Matching, a `hint` of `"boolean"` can be used to avoid the costly allocation of
    an _Iterable_ object when the return value won't be used for further matching (such as with `x is C`), while a `hint`
    of `"list"` indicates further matching will be performed on the result (such as with `x is C(1, 2)`).
  - For destructuring and binding patterns (e.g., this proposal), `hint` will always be `"list"`.

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
  when (Message.Move({ x, y })): ...;
}
```

This is even more evident with respect to complex, nested patterns:

```js
match (opt) {
  // without extractors
  when (${Option.Some} with [${Message.Move} with { x, y }]): ...;
  when (${Option.Some} with [${Message.Write} with [text]]): ...;

  // with extractors
  when (Option.Some(Message.Move({ x, y }))): ...;
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
  Move({x, y}),
  Write(message),
  ChangeColor(r, g, b),
}

// construction
const msg1 = Message.Move({ x: 10, y: 10 });
const msg2 = Message.Write("Hello");
const msg3 = Message.ChangeColor(0x00, 0xff, 0xff);

// destructuring
const Message.Move({x, y}) = msg1;          // x: 10, y: 10
const Message.Write(message) = msg2;        // message: "Hello"
const Message.ChangeColor(r, g, b) = msg3;  // r: 0, g: 255, b: 255

// pattern matching
match (msg) {
  when Message.Move({x, y}): ...;
  when Message.Write(message): ...;
  when Message.ChangeColor(r, g, b): ...;
  when Message.Quit: ...;
}
```

Here, declaration, construction, destructuring, and pattern matching are consistent for ADT enum members and values:

```js
enum Message of ADT { Move({ x, y }) }      // declaration
const msg =   Message.Move({ x, y });       // construction
const         Message.Move({ x, y }) = msg; // destructuring
match (msg) {
  when        Message.Move({ x, y }): ...;  // pattern matching
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

As noted before, it is possible that a future property-literal construction syntax that might be used by Algebraic Data
Types or other constructors. Adding a matching syntax for object extraction would be the responsibility of that proposal
and is out of scope for this proposal.


# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [x] Illustrative [examples][Examples] of usage.
* [x] High-level [API][API].

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].
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
