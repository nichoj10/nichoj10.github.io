---
title: Efficient Alias of a Built-In Emacs Lisp Function
layout: post
date: 2019-12-10T02:32:04Z
tags: [emacs, elisp, optimization]
uuid: 15421609-2681-4b75-99b2-b2d6aaa835fe
---

Suppose you don't like the names `car` and `cdr`, the traditional
identifiers for two halves of a lisp cons cell. [This is
misguided.][irreal] A cons is really just a 2-tuple, and the halves
don't have any particular meaning on their own, even as "head" and
"tail." However, maybe this is really important to you so you want to
do it anyway. What's the best way to go about it?

### defalias

Emacs Lisp has a built-in function just for this, `defalias`, which
is the obvious choice.

```cl
(defalias 'car-alias #'car)
```

The `car` built-in function is so fundamental to the language that [it
gets its own byte-code opcode][bc]. When you call `car` in your code,
the byte-compiler doesn't generate a function call, but instead uses a
single instruction. For example, here's an `add` function that sums
the `car` of its two arguments. I've followed the definition with its
disassembly (Emacs 26.3, [lexical scope][lex]):

```cl
(defun add (a b)
  (+ (car a) (car b)))
;; 0       stack-ref 1
;; 1       car
;; 2       stack-ref 1
;; 3       car
;; 4       plus
;; 5       return
```

There are zero function calls because of the dedicated `car` opcode, and
it has the optimal six byte-code instructions.

The problem with `defalias` is that the definition is permitted change
— or [be advised][advice] — and that robs the byte-compiler of
optimization opportunities. It's [a constraint][knife]. When the
byte-code compiler sees `car-alias`, it *must* emit a function call:

```cl
(defun add-alias (a b)
  (+ (car-alias a) (car-alias b)))
;; 0       constant  car-alias
;; 1       stack-ref 2
;; 2       call      1
;; 3       constant  car-alias
;; 4       stack-ref 2
;; 5       call      1
;; 6       plus
;; 7       return
```

This has two function calls and eight byte-code instructions. Those
function calls are significantly more expensive than a `car`
instruction, which will show in the benchmark later.

### defsubst

An alternative is `defsubst`, an inlined function definition, which
will inline an actual `car`. The semantics for `defsubst` are, like
macros, explicit that re-definitions may not affect previous uses, so
the constraint is gone. Unfortunately [the byte-code compiler is
pretty dumb][dumb], and does a poor job inlining `car-subst`.

```cl
(defsubst car-subst (x)
  (car x))

(defun add-subst (a b)
  (+ (car-subst a) (car-subst b)))
;; 0       stack-ref 1
;; 1       dup
;; 2       car
;; 3       stack-set 1
;; 5       stack-ref 1
;; 6       dup
;; 7       car
;; 8       stack-set 1
;; 10      plus
;; 11      return
```

There are zero function calls and ten byte-code instructions. The
`car` opcode *is* in use, but there are five unnecessary instructions.
This is still faster than making the function calls, though. If the
byte-code compiler was just a little smarter and could compile this to
the ideal case, then this would be the end of the discussion.

### cl-first

The built-in `cl-lib` package has a `cl-first` alias for `car`. This was
written by someone with intimate knowledge of Emacs Lisp, so how how
well did they do?

```cl
(require 'cl-lib)

(defun add-cl-first (a b)
  (+ (cl-first a) (cl-first b)))
;; 0       stack-ref 1
;; 1       car
;; 2       stack-ref 1
;; 3       car
;; 4       plus
;; 5       return
```

It's just like plain old `car`! How did they manage this? By using a
byte-compiler hint:

```cl
(defalias 'cl-first 'car)
(put 'cl-first 'byte-optimizer 'byte-compile-inline-expand)
```

They used `defalias`, but they also manually told the byte-compiler to
inline the definition like `defsubst`. In fact, `defsubst` expands to an
expression that sets `byte-compile-inline-expand`, but, as seen above,
the inline function overhead gets inlined and doesn't get eliminated.

### Benchmark

So how do the alternatives perform? ([benchmark source][gist])

    add           (0.594811299 0 0.0)
    add-alias     (1.232037132 0 0.0)
    add-subst     (0.700044324 0 0.0)
    add-cl-first  (0.58332882 0 0.0)

(The `car` of the list is the running time.) Since `add` and
`add-cl-first` have the same byte-codes, we shouldn't, and didn't, see
a significant difference. The simple use of `defalias` doubles the
running time, and using `defsubst` is about 15% slower.


[advice]: /blog/2013/01/22/
[bc]: /blog/2014/01/04/
[dumb]: /blog/2019/02/24/
[gist]: https://gist.github.com/skeeto/36baa3b1493f53eab4e082b449448a96
[irreal]: https://irreal.org/blog/?p=8500
[knife]: /blog/2019/12/09/
[lex]: /blog/2016/12/22/
