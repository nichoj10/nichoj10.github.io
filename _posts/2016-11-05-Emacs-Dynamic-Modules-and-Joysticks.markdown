---
title: Emacs, Dynamic Modules, and Joysticks
layout: post
date: 2016-11-05T04:01:51Z
tags: [emacs, elisp, c, linux]
uuid: c53305bb-4770-3a7f-934c-31eea37d38eb
---

Two months ago Emacs 25 was released and introduced a [new dynamic
module feature][tut]. Emacs can now load shared libraries built
against Emacs' module API, defined in [emacs-module.h][header]. What's
interesting about this API is that it doesn't require linking against
Emacs or any sort of library. Instead, at run time Emacs supplies the
module's initialization function with function pointers for the entire
API.

As a demonstration, in this article I'll build an Emacs joystick
interface (Linux only) using a dynamic module. It will allow Emacs to
read events from any joystick on the system. All the source code is
here:

* <https://github.com/skeeto/joymacs>

It includes a calibration interface (`M-x joydemo`) within Emacs:

[![](/img/joymacs/joymacs-thumb.png)](/img/joymacs/joymacs.png)

Currently, Emacs' emacs-module.h header is the entirety of the module
documentation. It's a bit thin and leaves ambiguities that requires
some reading of the Emacs source code. Even reading the source, it's
not clear which behaviors are a reliable part of the interface. For
example, if there's a pending non-local exit, it's safe for a function
to return `NULL` since the return value is never inspected (Emacs
25.1), but will this always be the case? While mistakes are
unforgiving (a hard crash), the API is mostly intuitive and it's been
pretty easy to feel my way around it.

*Update*: Philipp Stephani has [written thorough, reliable module
documentation][doc].

### Dynamic Module Types

All Emacs values — integers, floats, cons cells, vectors, strings,
etc. — are represented as the polymorphic, pointer-valued type,
`emacs_value`. Despite being a pointer, `NULL` is not a valid value,
as convenient as that would be. The API includes functions for
creating and extracting the fundamental types: integers, floats,
strings. Almost all other object types can only be accessed by making
Lisp function calls to regular Emacs functions from the module.

Modules also introduce a brand new Emacs object type: a *user
pointer*. These are [non-readable][read], opaque pointer values
returned by modules, typically representing a handle to some resource,
be it a memory block, database connection, or a joystick. These
objects include a finalizer function pointer — which, surprisingly, is
not permitted to be NULL — and their lifetime is managed by Emacs'
garbage collector.

User pointers are a somewhat dangerous feature since there's little to
stop Emacs Lisp code from misusing them. A Lisp program can take a
user pointer from one module and pass it to a function in a different
module. Since it's just a pointer, there's no way to type check it. At
best, a module could maintain a table of all its live pointers,
checking all user pointer arguments against the table before
dereferencing. But I don't expect this to be normal practice.

### Module Initialization

After loading the module through the platform's mechanism, the first
thing Emacs does is check for the symbol `plugin_is_GPL_compatible`.
While tacky, this is not surprising given the culture around Emacs.

Next it calls `emacs_module_init()`, passing it the first function
pointer. From this, the module can get a Lisp environment and start
doing Emacs things, such as binding module functions to Lisp symbols.

Here's a complete "Hello, world!" example:

~~~c
#include "emacs-module.h"

int plugin_is_GPL_compatible;

int
emacs_module_init(struct emacs_runtime *ert)
{
    emacs_env *env = ert->get_environment(ert);
    emacs_value message = env->intern(env, "message");
    const char hi[] = "Hello, world!";
    emacs_value string = env->make_string(env, hi, sizeof(hi) - 1);
    env->funcall(env, message, 1, &string);
    return 0;
}
~~~

In a real module, it's common to create function objects for native
functions, then fetch the `fset` symbol and make a Lisp call on it to
bind the newly-created function object to a name. You'll see this in
action later.

### Joystick API

The joystick API will closely resemble [Linux's own joystick API][js],
making for a fairly thin wrapper. It's so thin that Emacs *almost*
doesn't even need a dynamic module. This is because, on Linux,
joysticks are just files under `/dev/input/`. Want to see the input
events on the first joystick? Just read `/dev/input/js0`. So Plan 9.

Emacs already knows how to read files, but these virtual files are a
little *too* special for that. The header `linux/joystick.h` defines a
`struct js_event`:

~~~c
struct js_event {
    uint32_t time;  /* event timestamp in milliseconds */
    int16_t value;
    uint8_t type;
    uint8_t number; /* axis/button number */
};
~~~

The idea is to read from the joystick device into this structure. The
first several reads are initialization that define the axes and
buttons of the joystick and their initial state. Further events are
queued up for the file descriptor. This all means that the file can't
just be opened each time joystick input is needed. It has to be held
open for the duration, and is typically configured non-blocking.

The Emacs package will be called `joymacs` and there will be three
functions:

~~~cl
(joymacs-open N)
(joymacs-close JOYSTICK)
(joymacs-read JOYSTICK EVENT-VECTOR)
~~~

#### joymacs-open

The `joymacs-open` function will take an integer, opening the Nth
joystick (`/dev/input/jsN`). It will create a file descriptor for the
joystick device, returning it as a user pointer. Think of it as a sort
of "joystick handle." Now, it *could* instead return the file
descriptor as an integer, but the user pointer has two significant
benefits:

1. **The resource will be garbage collected.** If the caller loses
   track of a file descriptor returned as an integer, the joystick
   device will be held open until Emacs shuts down, using up one of
   Emacs' file descriptors. By putting it in a user pointer, the
   garbage collector will have the module to release the file
   descriptor if the user loses track of it.

2. **It should be difficult for the user to make a dangerous call.**
   Emacs Lisp can't create user pointers — they only come from modules
   — and so the module is less likely to get passed the wrong thing.
   In the case of `joystick-close`, the module will be calling
   `close(2)` on the argument. We definitely don't want to make that
   system call on file descriptors owned by Emacs. Further, since user
   pointers are mutable, the module can ensure it doesn't call
   `close(2)` twice.

Here's the implementation for `joymacs-open`. I'll over over each part
in detail.

~~~c
static emacs_value
joymacs_open(emacs_env *env, ptrdiff_t n, emacs_value *args, void *ptr)
{
    (void)ptr;
    (void)n;
    int id = env->extract_integer(env, args[0]);
    if (env->non_local_exit_check(env) != emacs_funcall_exit_return)
        return nil;
    char buf[64];
    int buflen = sprintf(buf, "/dev/input/js%d", id);
    int fd = open(buf, O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        emacs_value signal = env->intern(env, "file-error");
        emacs_value message = env->make_string(env, buf, buflen);
        env->non_local_exit_signal(env, signal, message);
        return nil;
    }
    return env->make_user_ptr(env, fin_close, (void *)(intptr_t)fd);
}
~~~

The C function name doesn't matter to Emacs. It's `static` because it
doesn't even matter if the function visible to Emacs. It will get the
function pointer later as part of initialization.

This is the prototype for all functions callable by Emacs Lisp,
regardless of its arity. It has four arguments:

1. It gets an environment, `env`, through which to call back into
   Emacs.

2. It gets `n`, the number of arguments. This is guaranteed to be the
   correct number of arguments, as specified later when creating the
   function object, so only variadic functions need to inspect this
   argument.

3. The Lisp arguments are passed as an array of values, `args`.
   There's no type declaration when declaring a function object, so
   these may be of the wrong type. I'll go over how to deal with this.

4. Finally, it gets an arbitrary pointer, supplied at function object
   creation time. This allows the module to create closures, but will
   usually be ignored.

The first thing the function does is extract its integer argument.
This is actually an `intmax_t`, but I don't think anyone has that many
USB ports. An `int` will suffice.

~~~c
    int id = env->extract_integer(env, args[0]);
    if (env->non_local_exit_check(env) != emacs_funcall_exit_return)
        return nil;
~~~

As for not underestimating fools, what if the user passed a value that
isn't an integer? Will the world come crashing down? Fortunately Emacs
checks that in `extract_integer` and, if there's a mismatch, sets a
pending error signal in the environment. This is really great because
checking types directly in the module is a *real pain the ass*. So,
before committing to anything further, such as opening a file, I check
for this signal and bail out early if necessary. In Emacs 25.1 it's
safe to return NULL since the return value will be completely ignored,
but I'd rather hedge my bets.

By the way, the `nil` here is a global variable set in initialization.
You don't just get that for free!

The next step is opening the joystick device, read-only and
non-blocking. The non-blocking is vital because the module would
otherwise hang Emacs later if there are no events (well, except for
the read being quickly interrupted by a POSIX signal).

~~~c
    char buf[64];
    int buflen = sprintf(buf, "/dev/input/js%d", id);
    int fd = open(buf, O_RDONLY | O_NONBLOCK);
~~~

If the joystick fails to open (e.g. it doesn't exist, or the user
lacks permission), manually set an error signal for a non-local exit.
I chose the `file-error` signal and I'm just using the filename as the
signal data.

~~~c
    if (fd == -1) {
        emacs_value signal = env->intern(env, "file-error");
        emacs_value message = env->make_string(env, buf, buflen);
        env->non_local_exit_signal(env, signal, message);
        return nil;
    }
~~~

Otherwise create the user pointer. No need to allocate any memory;
just stuff it in the pointer itself. If the user mistakenly passes it
to another module, it will sure be in for a surprise when it tries to
dereference it.

~~~c
    return env->make_user_ptr(env, fin_close, (void *)(intptr_t)fd);
~~~

The `fin_close()` function is defined as:

~~~c
static void
fin_close(void *fdptr)
{
    int fd = (intptr_t)fdptr;
    if (fd != -1)
        close(fd);
}
~~~

The garbage collector will call this function when the user pointer is
lost. If the user closes it early with `joymacs-close`, that function
will set the user pointer to -1, an invalid file descriptor, so that
it doesn't get closed a second time here.

#### joymacs-close

Here's `joymacs-close`, which is a bit simpler.

~~~c
static emacs_value
joymacs_close(emacs_env *env, ptrdiff_t n, emacs_value *args, void *ptr)
{
    (void)ptr;
    (void)n;
    int fd = (intptr_t)env->get_user_ptr(env, args[0]);
    if (env->non_local_exit_check(env) != emacs_funcall_exit_return)
        return nil;
    if (fd != -1) {
        close(fd);
        env->set_user_ptr(env, args[0], (void *)(intptr_t)-1);
    }
    return nil;
}
~~~

Again, it starts by extracting its argument, relying on Emacs to do
the check:

~~~c
    int fd = (intptr_t)env->get_user_ptr(env, args[0]);
    if (env->non_local_exit_check(env) != emacs_funcall_exit_return)
        return nil;
~~~

If the user pointer hasn't been closed yet, then close it and strip
out the file descriptor to prevent further closes.

~~~c
    if (fd != -1) {
        close(fd);
        env->set_user_ptr(env, args[0], (void *)(intptr_t)-1);
    }
~~~

#### joymacs-read

The `joymacs-read` function is doing something a little unusual for an
Emacs Lisp function. It takes two arguments: the joystick handle and a
5-element vector. Instead of returning the event in some
representation, it fills the vector with the event details. The are
two reasons for this:

1. The API has no function for creating vectors … though the module
   *could* get the `make-symbol` vector and call it to create a
   vector.

2. The idiom for event pumps is for the caller to supply a buffer to
   the pump. This has better performance by avoiding lots of
   unnecessary allocations, especially since events tend to be
   message-like objects with a short, well-defined extent.

Here's the full definition:

~~~c
static emacs_value
joymacs_read(emacs_env *env, ptrdiff_t n, emacs_value *args, void *ptr)
{
    (void)n;
    (void)ptr;
    int fd = (intptr_t)env->get_user_ptr(env, args[0]);
    if (env->non_local_exit_check(env) != emacs_funcall_exit_return)
        return nil;
    struct js_event e;
    int r = read(fd, &e, sizeof(e));
    if (r == -1 && errno == EAGAIN) {
        /* No more events. */
        return nil;
    } else if (r == -1) {
        /* An actual read error (joystick unplugged, etc.). */
        emacs_value signal = env->intern(env, "file-error");
        const char *error = strerror(errno);
        size_t len = strlen(error);
        emacs_value message = env->make_string(env, error, len);
        env->non_local_exit_signal(env, signal, message);
        return nil;
    } else {
        /* Fill out event vector. */
        emacs_value v = args[1];
        emacs_value type = e.type & JS_EVENT_BUTTON ? button : axis;
        emacs_value value;
        if (type == button)
            value = e.value ? t : nil;
        else
            value =  env->make_float(env, e.value / (double)INT16_MAX);
        env->vec_set(env, v, 0, env->make_integer(env, e.time));
        env->vec_set(env, v, 1, type);
        env->vec_set(env, v, 2, value);
        env->vec_set(env, v, 3, env->make_integer(env, e.number));
        env->vec_set(env, v, 4, e.type & JS_EVENT_INIT ? t : nil);
        return args[1];
    }
}
~~~

As before, extract the first argument and check for a signal. Then
call `read(2)` to get an event. If the read fails with `EAGAIN`, it's
not a real failure. There are just no more events, so return nil.

~~~c
    struct js_event e;
    int r = read(fd, &e, sizeof(e));
    if (r == -1 && errno == EAGAIN) {
        /* No more events. */
        return nil;
    }
~~~

If the read failed with something else — perhaps the joystick was
unplugged — signal an error. The `strerror(3)` string is used for the
signal data.

~~~c
    if (r == -1) {
        /* An actual read error (joystick unplugged, etc.). */
        emacs_value signal = env->intern(env, "file-error");
        const char *error = strerror(errno);
        emacs_value message = env->make_string(env, error, strlen(error));
        env->non_local_exit_signal(env, signal, message);
        return nil;
    }
~~~


Otherwise fill out the event vector. If the second argument isn't a
vector, or if it's too short, the signal will automatically get raised
by Emacs. The module can keep plowing through the `vec_set()` calls
safely since it's not committing to anything.

~~~c
        /* Fill out event vector. */
        emacs_value v = args[1];
        emacs_value type = e.type & JS_EVENT_BUTTON ? button : axis;
        emacs_value value;
        if (type == button)
            value = e.value ? t : nil;
        else
            value =  env->make_float(env, e.value / (double)INT16_MAX);
        env->vec_set(env, v, 0, env->make_integer(env, e.time));
        env->vec_set(env, v, 1, type);
        env->vec_set(env, v, 2, value);
        env->vec_set(env, v, 3, env->make_integer(env, e.number));
        env->vec_set(env, v, 4, e.type & JS_EVENT_INIT ? t : nil);
        return args[1];
~~~

The Linux event struct has four fields and the function fills out five
values of the vector. This is because the `type` field has a bit flag
indicating initialization events. This is split out into an extra
t/nil value. It also normalizes axis values and converts button values
into t/nil, which makes more sense for Emacs Lisp. The event itself is
returned since it's a truthy value and it's convenient for the caller.

The astute programmer might notice that the negative side of the axis
could go just below -1.0, since `INT16_MIN` has one extra value over
`INT16_MAX` (two's complement). It doesn't seem to be documented, but
the joystick drivers I've seen never exactly return `INT16_MIN`, so
this is in fact the correct way to normalize it.

#### Initialization

*Update 2021*: In a previous version of this article, I talked about
interning symbols during initialziation so that they do not need to be
re-interned each time the module is called. This [no longer works][bug],
and it was probably never intended to be work in the first place. The
lesson is simple: **Do not reuse Emacs objects between module calls.**

First grab the `fset` symbol since this function will be needed to bind
names to the module's functions.

~~~c
    emacs_value fset = env->intern(env, "fset");
~~~

Using `fset`, bind the functions. The second and third arguments to
`make_function` are the minimum and maximum number of arguments, which
[may look familiar][internals]. The last argument is that closure pointer
I mentioned at the beginning.

~~~c
    emacs_value args[2];
    args[0] = env->intern(env, "joymacs-open");
    args[1] = env->make_function(env, 1, 1, joymacs_open, doc, 0);
    env->funcall(env, fset, 2, args);
~~~

If the module is to be loaded with `require` like any other package,
it needs to provide: `(provide 'joymacs)`.

~~~c
    emacs_value provide = env->intern(env, "provide");
    emacs_value joymacs = env->intern(env, "joymacs");
    env->funcall(env, provide, 1, &joymacs);
~~~

And that's it!

The source repository now includes a port to Windows (XInput). If
you're on Linux or Windows, have Emacs 25 with modules enabled, and a
joystick is plugged in, then `make run` in the repository should bring
up Emacs running a joystick calibration demonstration. The module
can't poke at Emacs when events are ready, so instead there's a timer
that polls the module for events.

I'd like to someday see an Emacs Lisp game well-suited for a joystick.


[bug]: https://github.com/skeeto/joymacs/issues/1
[doc]: https://phst.github.io/emacs-modules
[header]: http://git.savannah.gnu.org/cgit/emacs.git/tree/src/emacs-module.h?h=emacs-25.1
[internals]: /blog/2014/01/04/
[js]: https://www.kernel.org/doc/Documentation/input/joystick-api.txt
[read]: /blog/2013/12/30/
[tut]: http://diobla.info/blog-archive/modules-tut.html
