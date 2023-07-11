---
title: How to Write Fast(er) Emacs Lisp
layout: post
date: 2017-01-30T21:08:19Z
tags: [emacs, elisp, optimization]
uuid: cee07e3d-08cc-3465-1a29-c1e30b5bd0e2
---

Not everything written in Emacs Lisp needs to be fast. Most of Emacs
itself — around 82% — is written in Emacs Lisp *because* those parts
are generally not performance-critical. Otherwise these functions
would be built-ins written in C. Extensions to Emacs don't have a
choice and — outside of a few exceptions like [dynamic modules][joy]
and inferior processes — must be written in Emacs Lisp, including
their performance-critical bits. Common performance hot spots are
automatic indentation, [AST parsing][js2], and [interactive
completion][jit].

Here are 5 guidelines, each very specific to Emacs Lisp, that will
result in faster code. The non-intrusive guidelines could be applied
at all times as a matter of style — choosing one equally expressive
and maintainable form over another just because it performs better.

There's one caveat: These guidelines are focused on Emacs 25.1 and
"nearby" versions. Emacs is constantly evolving. Changes to the
[virtual machine][vsm] and byte-code compiler may transform
currently-slow expressions into fast code, obsoleting some of these
guidelines. In the future I'll add notes to this article for anything
that changes.

### (1) Use lexical scope

This guideline refers to the following being the first line of every
Emacs Lisp source file you write:

~~~cl
;;; -*- lexical-binding: t; -*-
~~~

This point is worth mentioning again and again. Not only will [your
code be more correct][lex], it will be measurably faster. Dynamic
scope is still opt-in through the explicit use of *special variables*,
so there's absolutely no reason not to be using lexical scope. If
you've written clean, dynamic scope code, then switching to lexical
scope won't have any effect on its behavior.

Along similar lines, special variables are a lot slower than local,
lexical variables. Only use them when necessary.

### (2) Prefer built-in functions

Built-in functions are written in C and are, as expected,
significantly faster than the equivalent written in Emacs Lisp.
Complete as much work as possible inside built-in functions, even if
it might mean taking more conceptual steps overall.

For example, what's the fastest way to accumulate a list of items?
That is, new items go on the tail but, for algorithm reasons, the list
must be constructed from the head.

You might be tempted to keep track of the tail of the list, appending
new elements directly to the tail with `setcdr` (via `setf` below).

~~~cl
(defun fib-track-tail (n)
  (let* ((a 0)
         (b 1)
         (head (list 1))
         (tail head))
    (dotimes (_ n head)
      (psetf a b
             b (+ a b))
      (setf (cdr tail) (list b)
            tail (cdr tail)))))

(fib-track-tail 8)
;; => (1 1 2 3 5 8 13 21 34)
~~~

Actually, it's much faster to construct the list in reverse, then
destructively reverse it at the end.

~~~cl
(defun fib-nreverse (n)
  (let* ((a 0)
         (b 1)
         (list (list 1)))
    (dotimes (_ n (nreverse list))
      (psetf a b
             b (+ a b))
      (push b list))))
~~~

It might not look it, but `nreverse` is *very* fast. Not only is it a
built-in, it's got its own opcode. Using `push` in a loop, then
finishing with `nreverse` is the canonical and fastest way to
accumulate a list of items.

In `fib-track-tail`, the added complexity of tracking the tail in
Emacs Lisp is much slower than zipping over the entire list a second
time in C.

### (3) Avoid unnecessary lambda functions

I'm talking about `mapcar` and friends.

~~~cl
;; Slower
(defun expt-list (list e)
  (mapcar (lambda (x) (expt x e)) list))
~~~

Listen, I know you love [dash.el][dash] and higher order functions,
but *this habit ain't cheap*. The byte-code compiler does not know how
to inline these lambdas, so there's an additional per-element function
call overhead.

Worse, if you're using lexical scope like I told you, the above
example forms a *closure* over `e`. This means a new function object
is created (e.g. `make-byte-code`) each time `expt-list` is called. To
be clear, I don't mean that the lambda is recompiled each time — the
same byte-code string is shared between all instances of the same
lambda. A unique function vector (`#[...]`) and constants vector are
allocated and initialized each time `expt-list` is invoked.

Related mini-guideline: Don't create any more garbage than strictly
necessary in performance-critical code.

Compare to an implementation with an explicit loop, using the
`nreverse` list-accumulation technique.

~~~cl
(defun expt-list-fast (list e)
  (let ((result ()))
    (dolist (x list (nreverse result))
      (push (expt x e) result))))
~~~

* No unnecessary garbage is created.
* No unnecessary per-element function calls.

This is the fastest possible definition for this function, and it's
what you need to use in performance-critical code.

Personally I prefer the list comprehension approach, using `cl-loop`
from `cl-lib`.

~~~cl
(defun expt-list-fast (list e)
  (cl-loop for x in list
           collect (expt x e)))
~~~

The `cl-loop` macro will expand into essentially the previous
definition, making them practically equivalent. It takes some getting
used to, but writing efficient loops is a whole lot less tedious with
`cl-loop`.

In Emacs 24.4 and earlier, `catch`/`throw` is implemented by
converting the body of the `catch` into a lambda function and calling
it. If code inside the `catch` accesses a variable outside the `catch`
(very likely), then, in lexical scope, it turns into a closure,
resulting in the garbage function object like before.

In Emacs 24.5 and later, the byte-code compiler uses a new opcode,
`pushcatch`. It's a whole lot more efficient, and there's no longer a
reason to shy away from `catch`/`throw` in performance-critical code.
This is important because it's often the only way to perform an early
bailout.

### (4) Prefer using functions with dedicated opcodes

When following the guideline about using built-in functions, you might
have several to pick from. Some built-in functions have dedicated
virtual machine opcodes, making them much faster to invoke. Prefer
these functions when possible.

How can you tell when a function has an assigned opcode? Take a peek
at the `byte-defop` listings in [bytecomp.el][bc]. Optimization often
involves getting into the weeds, so don't be shy.

For example, the `assq` and `assoc` functions search for a matching
key in an association list (alist). Both are built-in functions, and
the only difference is that the former compares keys with `eq` (e.g.
symbol or integer keys) and the latter with `equal` (typically string
keys). The difference in performance between `eq` and `equal` isn't as
important as another factor: `assq` has its own opcode (158).

This means in performance-critical code you should prefer `assq`,
perhaps even going as far as restructuring your alists specifically to
have `eq` keys. That last step is probably a trade-off, which means
you'll want to make some benchmarks to help with that decision.

Another example is `eq`, `=`, `eql`, and `equal`. Some macros and
functions use `eql`, especially `cl-lib` which inherits `eql` as a
default from Common Lisp. Take `cl-case`, which is like `switch` from
the C family of languages. It compares elements with `eql`.

~~~cl
(defun op-apply (op a b)
  (cl-case op
    (:norm (+ (* a a) (* b b)))
    (:disp (abs (- a b)))
    (:isin (/ b (sin a)))))
~~~

The `cl-case` expands into a `cond`. Since Emacs byte-code lacks
support for jump tables, there's not much room for cleverness.

**Update**: Emacs 26.1, released May 2018, introduced a jump table
opcode.

~~~cl
(defun op-apply (op a b)
  (cond
   ((eql op :norm) (+ (* a a) (* b b)))
   ((eql op :disp) (abs (- a b)))
   ((eql op :isin) (/ b (sin a)))))
~~~

It turns out `eql` is pretty much always the worst choice for
`cl-case`. Of the four equality functions I listed, the only one
lacking an opcode is `eql`. A faster definition would use `eq`. (In
theory, `cl-case` *could* have done this itself because it knows all
the keys are symbols.)

~~~cl
(defun op-apply (op a b)
  (cond
   ((eq op :norm) (+ (* a a) (* b b)))
   ((eq op :disp) (abs (- a b)))
   ((eq op :isin) (/ b (sin a)))))
~~~

Fortunately `eq` can safely compare integers in Emacs Lisp. You only
need `eql` when comparing symbols, integers, and floats all at once,
which is unusual.

### (5) Unroll loops using and/or

Consider the following function which checks its argument against a
list of numbers, bailing out on the first match. I used `%` instead of
`mod` since the former has an opcode (166) and the latter does not.

~~~cl
(defun detect (x)
  (catch 'found
    (dolist (f '(2 3 5 7 11 13 17 19 23 29 31))
      (when (= 0 (% x f))
        (throw 'found f)))))
~~~

The byte-code compiler doesn't know how to unroll loops. Fortunately
that's something we can do for ourselves using `and` and `or`. The
compiler will turn this into clean, efficient jumps in the byte-code.

~~~cl
(defun detect-unrolled (x)
  (or (and (= 0 (% x 2)) 2)
      (and (= 0 (% x 3)) 3)
      (and (= 0 (% x 5)) 5)
      (and (= 0 (% x 7)) 7)
      (and (= 0 (% x 11)) 11)
      (and (= 0 (% x 13)) 13)
      (and (= 0 (% x 17)) 17)
      (and (= 0 (% x 19)) 19)
      (and (= 0 (% x 23)) 23)
      (and (= 0 (% x 29)) 29)
      (and (= 0 (% x 31)) 31)))
~~~

In Emacs 24.4 and earlier with the old-fashioned lambda-based `catch`,
the unrolled definition is seven times faster. With the faster
`pushcatch`-based `catch` it's about twice as fast. This means the
loop overhead accounts for about half the work of the first definition
of this function.

Update: It was pointed out in the comments that this particular
example is equivalent to a `cond`. That's literally true all the way
down to the byte-code, and it would be a clearer way to express the
unrolled code. In real code it's often not *quite* equivalent.

Unlike some of the other guidelines, this is certainly something you'd
only want to do in code you know for sure is performance-critical.
Maintaining unrolled code is tedious and error-prone.

I've had the most success with this approach by not by unrolling these
loops myself, but by [using a macro][dsl], or [similar][jit], to
generate the unrolled form.

~~~cl
(defmacro with-detect (var list)
  (cl-loop for e in list
           collect `(and (= 0 (% ,var ,e)) ,e) into conditions
           finally return `(or ,@conditions)))

(defun detect-unrolled (x)
  (with-detect x (2 3 5 7 11 13 17 19 23 29 31)))
~~~

### How can I find more optimization opportunities myself?

Use `M-x disassemble` to inspect the byte-code for your own hot spots.
Observe how the byte-code changes in response to changes in your
functions. Take note of the sorts of forms that allow the byte-code
compiler to produce the best code, and then exploit it where you can.


[jit]: /blog/2016/12/11/
[lex]: /blog/2016/12/22/
[dsl]: /blog/2016/12/27/
[joy]: /blog/2016/11/05/
[js2]: https://github.com/mooz/js2-mode
[vsm]: /blog/2014/01/04/
[dash]: https://github.com/magnars/dash.el
[bc]: https://github.com/emacs-mirror/emacs/blob/master/lisp/emacs-lisp/bytecomp.el
