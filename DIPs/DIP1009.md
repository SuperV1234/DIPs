# Value closures

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | placeholder                                                     |
| RC#             | 0                                                               |
| Author:         | Vittorio Romeo (vittorio.romeo@outlook.com)                     |
| Implementation: |                                                                 |
| Status:         |                                                                 |

## Abstract

Currently [`FunctionLiteral`](https://dlang.org/spec/expression.html#FunctionLiteral)s that capture their outer context *(i.e. closures/delegates)* require an allocation and the garbage collector. This can incur unnecessary performance degradation and disallows the use of `FunctionLiteral`s *(most notably lambdas)* in `@nogc` code.

This paper proposes a new `ValueFunctionLiteral` expression family that produces closures with value semantics. The user explicitly specifies what variables from the outer scope must be "captured" and how. `ValueFunctionLiteral` would be just syntactic sugar for [structs](https://dlang.org/spec/struct.html).

### Links

* [D forum: **"Value closures (no GC allocation)"**](https://forum.dlang.org/post/lrtwpeyifchntuzxccyt@forum.dlang.org)

## Description

TODO:

* Detailed technical description of the new semantics.
* Language grammar changes (per https://dlang.org/spec/grammar.html).
* Similarity/differences with C++.
* Capture modes/syntax.
* Just syntactic sugar for struct generation.
* Comparison vs library solutions.

### Rationale

TODO:

State a short motivation about the importance and benefits of the proposed
change.  An existing, well-known issue or a use case for an existing projects
can greatly increase the chances of the DIP being understood and carefully
evaluated.

### Breaking changes / deprecation process

This paper proposes a pure language extension: no existing code will become invalid.
*This is arguable. We will be proposing a syntax redundant w.r.t. current behavior (i.e. capture by reference), so maybe we should consider proposing deprecation?*

### Examples

#### Captureless

```d
void foo() @nogc
{
    auto l = []() => writeln("hello!");
    // or simply, using existing syntax:
    //auto l = () => writeln("hello!");
}
```

...would roughly expand to:

```d
void foo() @nogc
{
    struct anonymous_l
    {
        void opCall() { writeln("hello!"); }
    }

    anonymous_l l;
}
```

#### Capture lvalue by copying

```d
void foo() @nogc
{
    int i = 10;
    auto l = [i]() => writeln(i);
}
```

...would roughly expand to (names are purely illustraitve):

```d
void foo() @nogc
{
    struct anonymous_l
    {
        int i;
        this(int i) { this.i = i; }
        void opCall() { writeln(i); }
    }

    int i = 10;
    auto l = anonymous_l(i);
}
```

#### Capture rvalue by moving

```d
struct NonCopyable
{
    int i;
    @disable this(this);
}

void foo() @nogc
{
    auto l = [n = NonCopyable(42)]() => n.i;
}
```

...would roughly expand to (names are purely illustraitve):

```d
void foo() @nogc
{
    struct anonymous_l
    {
        NonCopyable n;
        this(NonCopyable n) { this.n = __compiler_move(n); } // i.e. compiler constructs n in-place
        auto opCall() { return n.i; }
    }
    
    auto l = anonymous_l(NonCopyable(42));
}
```

#### Capture by reference

```d
void foo() @nogc
{
    int i = 10;
    auto l = [ref i]() => writeln(i);
}
```

...would roughly expand to (names are purely illustraitve):

```d
void foo() @nogc
{
    struct anonymous_l
    {
        ref int _i;
        this(ref int i) { _i = i; }
        auto opCall() { writeln("hello!"); }
    }

    int i = 10;
    auto l = anonymous_l(i);
}
```
TODO: propose deprecating current "always-by-reference" approach in favor of explicit by-reference capture. "scope" keyword should be used to determine whether a dynamic allocation is required for by-reference captures.

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

Will contain comments / requests from language authors once review is complete,
filled out by the DIP manager - can be both inline and linking to external
document.
