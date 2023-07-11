---
title: Emacs Lisp Lambda Expressions Are Not Self-Evaluating
layout: post
date: 2018-02-22T21:30:57Z
tags: [emacs, elisp, lang]
uuid: 7a3cd1d1-a48c-3c9b-1564-eacde8b9aa4d
---

This week I made a mistake that ultimately enlightened me about the
nature of function objects in Emacs Lisp. There are three kinds of
function objects, but they each behave very differently when evaluated
as objects.

But before we get to that, let's talk about one of Emacs'
embarrassing, old missteps: `eval-after-load`.

### Taming an old dragon

One of the long-standing issues with Emacs is that loading Emacs Lisp
files (.el and .elc) is a slow process, even when those files have
been byte compiled. There are a number of dirty hacks in place to deal
with this issue, and the biggest and nastiest of them all is the
[*dumper*][dumper], also known as *unexec*.

The Emacs you routinely use throughout the day is actually a previous
instance of Emacs that's been resurrected from the dead. Your undead
Emacs was probably created months, if not years, earlier, back when it
was originally compiled. The first stage of compiling Emacs is to
compile a minimal C core called `temacs`. The second stage is loading
a bunch of Emacs Lisp files, then dumping a memory image in an
unportable, platform-dependent way. On Linux, this actually [requires
special hooks in glibc][glibc]. The Emacs you know and love is this
dumped image loaded back into memory, continuing from where it left
off just after it was compiled. Regardless of your own feelings on the
matter, you have to admit [this *is* a very lispy thing to do][bs].

There are two notable costs to Emacs' dumper:

1. The dumped image contains hard-coded memory addresses. This means
   Emacs can't be a *Position Independent Executable* (PIE). It can't
   take advantage of a security feature called *Address Space Layout
   Randomization* (ASLR), which would increase the difficulty of
   [exploiting][exploit] some [classes of bugs][abort]. This might be
   important to you if Emacs processes untrusted data, such as when it's
   used as [a mail client][mail], [a web server][web] or generally
   [parses data downloaded across the network][elfeed].

2. It's not possible to cross-compile Emacs since it can only be dumped
   by running `temacs` on its target platform. As an experiment I've
   attempted to dump the Windows version of Emacs on Linux using
   [Wine][wine], but was unsuccessful.

The good news is that there's [a portable dumper][port] in the works
that makes this a lot less nasty. If you're adventurous, you can
already disable dumping and run `temacs` directly by setting
[`CANNOT_DUMP=yes` at compile time][cd]. Be warned, though, that a
non-dumped Emacs takes several seconds, or worse, to initialize
*before* it even begins loading your own configuration. It's also
somewhat buggy since it seems nobody ever runs it this way
productively.

The other major way Emacs users have worked around slow loading is
aggressive use of lazy loading, generally via *autoloads*. The major
package interactive entry points are defined ahead of time as stub
functions. These stubs, when invoked, load the full package, which
overrides the stub definition, then finally the stub re-invokes the
new definition with the same arguments.

To further assist with lazy loading, an evaluated `defvar` form will
not override an existing global variable binding. This means you can,
to a certain extent, configure a package before it's loaded. The
package will not clobber any existing configuration when it loads.
This also explains the bizarre interfaces for the various hook
functions, like `add-hook` and `run-hooks`. These accept symbols — the
*names* of the variables — rather than *values* of those variables as
would normally be the case. The `add-to-list` function does the same
thing. It's all intended to cooperate with lazy loading, where the
variable may not have been defined yet.

#### eval-after-load

Sometimes this isn't enough and you need some some configuration to
take place after the package has been loaded, but without forcing it
to load early. That is, you need to tell Emacs "evaluate this code
after this particular package loads." That's where `eval-after-load`
comes into play, except for its fatal flaw: it takes the word "eval"
completely literally.

The first argument to `eval-after-load` is the name of a package. Fair
enough. The second argument is a form that will be passed to `eval`
after that package is loaded. Now hold on a minute. The general rule
of thumb is that if you're calling `eval`, you're probably doing
something seriously wrong, and this function is no exception. This is
*completely* the wrong mechanism for the task.

The second argument should have been a function — either a (sharp
quoted) symbol or a function object. And then instead of `eval` it
would be something more sensible, like `funcall`. Perhaps this
improved version would be named `call-after-load` or `run-after-load`.

The big problem with passing an s-expression is that it will be left
uncompiled due to being quoted. [I've talked before about the
importance of evaluating your lambdas][lambda]. `eval-after-load` not
only encourages badly written Emacs Lisp, it demands it.

```cl
;;; BAD!
(eval-after-load 'simple-httpd
                 '(push '("c" . "text/plain") httpd-mime-types))
```

This was all corrected in Emacs 25. If the second argument to
`eval-after-load` is a function — the result of applying `functionp` is
non-nil — then it uses `funcall`. There's also a new macro,
`with-eval-after-load`, to package it all up nicely.

```cl
;;; Better (Emacs >= 25 only)
(eval-after-load 'simple-httpd
  (lambda ()
    (push '("c" . "text/plain") httpd-mime-types)))

;;; Best (Emacs >= 25 only)
(with-eval-after-load 'simple-httpd
  (push '("c" . "text/plain") httpd-mime-types))
```

Though in both of these examples the compiler will likely warn about
`httpd-mime-types` not being defined. That's a problem for another
day.

#### A workaround

But what if you *need* to use Emacs 24, as was the [situation that
sparked this article][issue]? What can we do with the bad version of
`eval-after-load`? We could situate a lambda such that it's evaluated,
but then smuggle the resulting function object into the form passed to
`eval-after-load`, all using a backquote.

```cl
;;; Note: this is subtly broken
(eval-after-load 'simple-httpd
  `(funcall
    ,(lambda ()
       (push '("c" . "text/plain") httpd-mime-types)))
```

When everything is compiled, the backquoted form evalutes to this:

```cl
(funcall #[0 <bytecode> [httpd-mime-types ("c" . "text/plain")] 2])
```

Where the second value (`#[...]`) is a [byte-code object][int].
However, as the comment notes, this is subtly broken. A cleaner and
correct way to solve all this is with a named function. The damage
caused by `eval-after-load` will have been (mostly) minimized.

```cl
(defun my-simple-httpd-hook ()
  (push '("c" . "text/plain") httpd-mime-types))

(eval-after-load 'simple-httpd
  '(funcall #'my-simple-httpd-hook))
```

But, let's go back to the anonymous function solution. What was broken
about it? It all has to do with evaluating function objects.

### Evaluating function objects

So what happens when we evaluate an expression like the one above with
`eval`? Here's what it looks like again.

```cl
(funcall #[...])
```

First, `eval` notices it's been given a non-empty list, so it's probably
a function call. The first argument is the name of the function to be
called (`funcall`) and the remaining elements are its arguments. *But*
each of these elements must be evaluated first, and the *result* of that
evaluation becomes the arguments.

Any value that isn't a list or a symbol is *self-evaluating*. That is,
it evaluates to its own value:

```cl
(eval 10)
;; => 10
```

If the value is a symbol, it's treated as a variable. If the value is a
list, it goes through the function call process I'm describing (or one
of a number of other special cases, such as macro expansion, lambda
expressions, and special forms).

So, conceptually `eval` recurses on the function object `#[...]`. A
function object is not a list or a symbol, so it's self-evaluating. No
problem.

```cl
;; Byte-code objects are self-evaluating

(let ((x (byte-compile (lambda ()))))
  (eq x (eval x)))
;; => t
```

What if this code *wasn't* compiled? Rather than a byte-code object,
we'd have some other kind of function object for the interpreter.
Let's examine the dynamic scope (*shudder*) case. Here, a lambda
*appears* to evaluate to itself, but appearances can be deceiving:

```cl
(eval (lambda ())
;; => (lambda ())
```

However, this is not self-evaluation. **Lambda expressions are not
self-evaluating**. It's merely *coincidence* that the result of
evaluating a lambda expression looks like the original expression.
This is just how the Emacs Lisp interpreter is currently implemented
and, strictly speaking, it's an implementation detail that *just so
happens* to be mostly compatible with byte-code objects being
self-evaluating. It would be a mistake to rely on this.

Instead, **dynamic scope lambda expression evaluation is
[idempotent][idempotent].** Applying `eval` to the result will return
an `equal`, but not identical (`eq`), expression. In contrast, a
self-evaluating value is also idempotent under evaluation, but with
`eq` results.

```cl
;; Not self-evaluating:

(let ((x '(lambda ())))
  (eq x (eval x)))
;; => nil

;; Evaluation is idempotent:

(let ((x '(lambda ())))
  (equal x (eval x)))
;; => t

(let ((x '(lambda ())))
  (equal x (eval (eval x))))
;; => t
```

So, with dynamic scope, the subtly broken backquote example will still
work, but only by sheer luck. Under lexical scope, the situation isn't
so lucky:

```cl
;;; -*- lexical-scope: t; -*-

(lambda ())
;; => (closure (t) nil)
```

These interpreted lambda functions are neither self-evaluating nor
idempotent. Passing `t` as the second argument to `eval` tells it to
use lexical scope, as shown below:

```cl
;; Not self-evaluating:

(let ((x '(lambda ())))
  (eq x (eval x t)))
;; => nil

;; Not idempotent:

(let ((x '(lambda ())))
  (equal x (eval x t)))
;; => nil

(let ((x '(lambda ())))
  (equal x (eval (eval x t) t)))
;; error: (void-function closure)
```

I can [imagine an implementation][imagine] of Emacs Lisp where dynamic
scope lambda expressions are in the same boat, where they're not even
idempotent. For example:

```cl
;;; -*- lexical-binding: nil; -*-

(lambda ())
;; => (totally-not-a-closure ())
```

Most Emacs Lisp would work just fine under this change, and only code
that makes some kind of logical mistake — where there's nested
evaluation of lambda expressions — would break. This essentially
already happened when lots of code was quietly switched over to
lexical scope after Emacs 24. Lambda idempotency was lost and
well-written code didn't notice.

There's a temptation here for Emacs to define a `closure` function or
special form that would allow interpreter closure objects to be either
self-evaluating or idempotent. This would be a mistake. It would only
serve as a hack that covers up logical mistakes that lead to nested
evaluation. Much better to catch those problems early.

### Solving the problem with one character

So how do we fix the subtly broken example? With a strategically
placed quote right before the comma.

```cl
(eval-after-load 'simple-httpd
  `(funcall
    ',(lambda ()
        (push '("c" . "text/plain") httpd-mime-types)))
```

So the form passed to `eval-after-load` becomes:

```cl
;; Compiled:
(funcall (quote #[...]))

;; Dynamic scope:
(funcall (quote (lambda () ...)))

;; Lexical scope:
(funcall (quote (closure (t) () ...)))
```

The quote prevents `eval` from evaluating the function object, which
would be either needless or harmful. There's also an argument to be
made that this is a perfect situation for a sharp-quote (`#'`), which
exists to quote functions.


[abort]: /blog/2012/09/28/
[bs]: /blog/2011/01/30/
[cd]: https://lists.gnu.org/archive/html/bug-gnu-emacs/2016-11/msg00729.html
[dumper]: https://lwn.net/Articles/707615/
[elfeed]: https://github.com/skeeto/elfeed
[exploit]: /blog/2017/07/19/
[glibc]: https://lwn.net/Articles/707615/
[idempotent]: https://labs.spotify.com/2013/06/18/creative-usernames/
[imagine]: /blog/2017/05/03/
[int]: /blog/2014/01/04/
[issue]: https://github.com/skeeto/elfeed/pull/268
[lambda]: /blog/2017/12/14/
[mail]: /blog/2013/09/03/
[port]: https://lists.gnu.org/archive/html/emacs-devel/2018-02/msg00347.html
[web]: https://github.com/skeeto/emacs-web-server
[wine]: https://www.winehq.org/
