References:

- <https://thephd.dev/_vendor/future_cxx/papers/C%20-%20Initialized%20const%20Integer%20Declarations.html>
- <https://twitter.com/__phantomderp/status/1693696514931986728>

The proposal is that a block-scope declaration of an integer object
with a `const` keyword and an initializer that's a integer constant
expression should be implicitly `constexpr`.

C++ already has a similar rule, introduced before C++ acquired the
`constexpr` keyword.

I submit that the proposed change is rendered unnecessary by the
introduction of the `constexpr` keyword (introduced in the same 2023
edition of the C standard for which this proposal is being made), and
that it would introduce more confusion about the meaning of `const`.

A minor point: Quoting the Abstract of the proposal:

> It asks that such `const` integer type declarations that are both
> declared and initialized in the same statement are implicitly made
> `constexpr`, thereby fulfilling the expectations of users.

This could be worded better.  The phrase "integer type declarations"
suggests declarations of integer types, which is clearly not the
intent.  The declarations being discussed are also definitions.
And the "same statement" is incorrect, since declarations (at least
in C) are not statements.

The first example in the proposal is:
```
int main () {
  const int n = 1 + 2;
  const int a[n];
  return sizeof(a);
}
```

(The `const` on the definition of `a` seems superfluous.  And I'd
drop the parentheses on the `sizeof`, but that's just me.)

Under current C rules, this is unambiguous.  `n` is not an integer
constant expression, and `a` is a VLA.

> Will `sizeof(a)` be executed at compile-time or will it be run at
> execution time (AKA run-time) and pull the value from somewhere in
> the binary?

A sufficiently clever compiler (gcc and clang certainly both qualify)
can evaluate some non-constant expressions at compile time.  The fact
that the generated code is the same as it would be if `n` were replaced
by `3` does not imply that the compiler is treating `a` as a VLA.
And in fact, both gcc and clang warn about the definition of `a` with
invoked with the `-Wvla` option:
```
$ gcc -c -std=c17 -pedantic-errors -Wvla c.c
c.c: In function ‘main’:
c.c:3:3: warning: ISO C90 forbids variable length array ‘a’ [-Wvla]
    3 |   const int a[n];
      |   ^~~~~
$ clang -c -std=c17 -pedantic-errors -Wvla c.c
c.c:3:15: warning: variable length array used [-Wvla]
  const int a[n];
              ^
1 warning generated.
$
```

And both reject it with `-std=c90 -pedantic-errors`.

Furthermore, it's clear that neither gcc nor clang treats `n` in the
example as an integer constant expression, as we can see by adding
an attempt to use it in a case label:
```
$ gcc -std=c17 -pedantic-errors -c c.c
c.c: In function ‘main’:
c.c:3:14: error: case label does not reduce to an integer constant
    3 |   switch (3) case n:;
      |              ^~~~
$ clang -std=c17 -pedantic-errors -c c.c
c.c:3:19: error: expression is not an integer constant expression; folding it to a constant is a GNU extension [-Werror,-Wgnu-folding-constant]
  switch (3) case n:;
                  ^
1 error generated.
$
```

Microsoft's Visual C compiler (Visual Studio Community 2022, Version
17.3.4) does not support VLAs.  With appropriate settings, it sets
`__STDC_VERSION__` to `201710L`, and `__STDC_NO_VLA__` to `1`.
Given the declaration `const int n = 1 + 2;`, it does not treat `n`
as a constant expression, and it rejects `const int a[n];` with 
"error C2057: expected constant expression".

> However, during National Body (NB) comment processing, an NB comment
> pointed out that there was a lot of code relying on the fact that this
> was being treated -- not just by the backend with its optimizations
> -- but by the frontend of many compilers to be a plain, compile-time
> C array

I'm not aware of any valid code whose behavior differs depending
on whether an array whose length is not a constant expression but
can be evaluated at compile time is a VLA or not, for a compiler that
supports VLAs.  A VLA with compile-time computable bounds is very
nearly equivalent to a non-VLA array, and identical code can be
generated for both.  Quite possibly I've missed something.  I'd like
to see some concrete examples (depending on the semantics of C code,
not on examining the generated machine or assembly code).

C23 introduces the `constexpr` keyword, borrowed from C++.  If I want
`a` to be a non-VLA array, I'll define `n` as `constexpr`, not as
`const`.  Or, if I don't have a C23 compiler, I'll use one of several
pre-`constexpr` tricks:

- `enum { n = 1 + 2 }; // works only for type int prior to C23`
- `#define N (1 + 2)`

And of course the proposed change does not help users of
pre-C23 compilers, unless some of them implement the change as a
(non-conforming) extension.

I've always found it somewhat awkward to explain to beginners what
`const` means in C.  The name makes it extremely tempting to conflate
`const` with "constant", but in fact (at least in current C),
`const` means "read-only", not "compile-time constant".  These are
very distinct concepts, and I oppose conflating them.  For example:

```
const int r = rand();
```
is perfectly valid, and
```
constexpr int r = rand();
```
is a constraint violation.

It's much easier to explain that `const` means "read-only", than
to explain that `const` means "read-only" *except* in certain cases
where it also means "compile-time constant".

As far as I can tell, existing C code *does not* rely on
`const`-defined integer objects being treated as compile-time
constants.  In the case of array bounds, some such code may
unintentionally rely on compiler support for VLAs.  If such code
needs to be compiled with an implementation that forbids VLAs, the
fix is to replace `const` by `constexpr`.

```
int main () {
  constexpr int n = 1 + 2;
  int a[n];
  return sizeof(a);
}
```

-- Keith Thompson <Keith.S.Thompson@gmail.com>
