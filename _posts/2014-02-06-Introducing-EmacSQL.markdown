---
title: Introducing EmacSQL
layout: post
date: 2014-02-06T05:52:37Z
tags: [emacs, elisp]
uuid: af878773-296e-3411-593a-cc516856b832
---

Yesterday I made the first official release of [EmacSQL][emacsql], an
Emacs package I've been working on for the past few weeks. EmacSQL is
a high-level SQL database for Emacs. It primarily targets SQLite as a
back-end, but it also currently supports PostgreSQL and MySQL.

 * [https://github.com/skeeto/emacsql](https://github.com/skeeto/emacsql)

It's [available on MELPA][melpa] and is ready for immediate use. It
depends on the [finalizers package][finalize] I added last week.

While there's a non-Elisp component, SQLite, there are no special
requirements for the user to worry about. When the package's Elisp is
compiled, if a C compiler is available it will use it to compile a
SQLite binary for EmacSQL. If not, it will later offer to download a
pre-built binary that I built. Ideally this makes the non-Elisp part
of EmacSQL completely transparent and users can pretend Emacs has a
built-in relational database.

The official SQLite command line shell is not used even if present,
and I'll explain why below.

Just as [Skewer][skewer] jump started my web development experience,
EmacSQL has been a crash course in SQL and relational databases.
Before starting this project I knew little about this topic and I've
gained a lot of appreciation for it in the process. Building an Emacs
extension is a very rapid way to dive into a new topic.

If you're a total newb about this stuff like I was and want to learn
SQL for SQLite yourself, I highly recommend [Using SQLite][book]. It's
a really solid introduction.

### High-level SQL Compiler

By "high-level" I mean that it goes beyond assembling strings
containing SQL code. In EmacSQL, statements are assembled from
s-expressions which, behind the scenes, are compiled into SQL using
some simple rules. This means if you already know SQL you should be
able to hit the ground running with EmacSQL. Here's an example,

~~~cl
(require 'emacsql)

;; Connect to the database, SQLite in this case:
(defvar db (emacsql-connect "~/office.db"))

;; Create a table with 3 columns:
(emacsql db [:create-table patients
             ([name (id integer :primary-key) (weight float)])])

;; Insert a few rows:
(emacsql db [:insert :into patients
             :values (["Jeff" 1000 184.2] ["Susan" 1001 118.9])])

;; Query the database:
(emacsql db [:select [name id]
             :from patients
             :where (< weight 150.0)])
;; => (("Susan" 1001))

;; Queries can be templates, using $s1, $i2, etc. as parameters:
(emacsql db [:select [name id]
             :from patients
             :where (> weight $s1)]
         100)
;; => (("Jeff" 1000) ("Susan" 1001))
~~~

A query is a vector of keywords, identifiers, parameters, and data.
Thanks to parameters, these s-expression statements should not need to
be constructed dynamically at run-time.

The compilation rules are listed in the EmacSQL documentation so I
won't repeat them in detail here. In short, lisp keywords become SQL
keywords, row-oriented information is always presented as vectors,
expressions are lists, and symbols are identifiers, except when
quoted.

~~~cl
[:select [name weight] :from patients :where (< weight 150.0)]
~~~

That compiles to this,

~~~sql
SELECT name, weight FROM patients WHERE weight < 150.0;
~~~

Also, any [readable lisp value][readable] can be stored in an
attribute. Integers are mapped to INTEGER, floats are mapped to REAL,
nil is mapped to NULL, and everything else is printed and stored as
TEXT. The specifics vary depending on the back-end.

#### Parameters

A symbol beginning with a dollar sign is a parameter. It has a type —
identifier (i), scalar (s), vector (v), schema (S) — and an argument
position.

~~~cl
[:select [$i1] :from $i2 :where (< $i3 $s4)]
~~~

Given the arguments `name people age 21`, three symbols and an
integer, it compiles to:

~~~cl
SELECT name FROM people WHERE age < 21;
~~~

A vector parameter refers to rows to be inserted or as a set for an
`IN` expression.

~~~cl
[:insert-into people [name age] :values $v1]
~~~

Given the argument `(["Jim" 45] ["Jeff" 34])`, a list of two rows,
this becomes,

~~~sql
INSERT INTO people (name, age) VALUES ('"Jim"', 45), ('"Jeff"', 34);
~~~

And this,

~~~cl
[:select * :from tags :where (in tag $v1)]
~~~

Given the argument `[hiking camping biking]` becomes,

~~~sql
SELECT * FROM tags WHERE tag IN ('hiking', 'camping', 'biking');
~~~

When writing these expressions keep in mind the command
`emacsql-show-last-sql`. It will display in the minibuffer the SQL
result of the s-expression statement before the point.

### Schemas

A table schema is a list whose first element is a column specification
vector (i.e. row-oriented information is presented as vectors). The
remaining elements are table constraints. Here are the examples from
the documentation,

~~~cl
;; No constraints schema with four columns:
([name id building room])

;; Add some column constraints:
([(name :unique) (id integer :primary-key) building room])

;; Add some table constraints:
([(name :unique) (id integer :primary-key) building room]
 (:unique [building room])
 (:check (> id 0)))
~~~

In the handful of EmacSQL databases I've created for practice and
testing, I've put the schema in a global constant. A table schema is a
part of a program's type specifications, and rows are instances of
that type, so it makes sense to declare schemas up top with things
like defstructs.

These schemas can be substituted into a SQL statement using a `$S`
parameter (capital "S" for *S*chema).

~~~cl
(defconst foo-schema-people
  '([(person-id integer :primary-key) name age]))

;; ...

(defun foo-init (db)
  (emacsql db [:create-table $i1 $S2] 'people foo-schema-people))
~~~

### Back-ends

Everything I've discussed so far is restricted to the SQL statement
compiler. It's completely independent of the back-end implementations,
themselves mostly handling strings of SQL statements.

#### SQLite Implementation Difficulties

A little over a year ago I wrote [a pastebin webapp][pastebin] in
Elisp. I wanted to use SQLite as a back-end for storing pastes but
struggled to get the SQLite command shell, sqlite3, to cooperate with
Emacs. The problem was that all of the output modes except for "tcl"
are ambiguous. This includes the "csv" formatted output. TEXT values
can dump newlines, allowing rows to span an arbitrary number of lines.
They can dump things that look like the sqlite3 prompt, so it's
impossible to know when sqlite3 is done printing results. I ultimately
decided the command shell was inadequate as an Emacs subprocess.

Recently there [was some discussion][elfeed] from alexbenjm and Andres
Ramirez on an Elfeed post about using SQLite as an Elfeed back-end.
This inspired me to take another look and that's when I came up with a
workaround for SQLite's ambiguity: only store printed Elisp values for
TEXT values! With `print-escape-newlines` set, TEXT values no longer
span multiple lines, and I can use `read` to pull in data from
sqlite3. All of sqlite3's output modes were now unambiguous.

However, after making significant progress I discovered an even bigger
issue: GNU Readline. The sqlite3 binary provided by Linux package
repositories is almost always compiled with Readline support. This
makes the tool much more friendly to use, but it's a huge problem for
Emacs.

First, sqlite3 the command shell is not up to the same standards as
SQLite the database. Not by a long shot. In my short time working with
SQLite I've already discovered several bugs in the command shell. For
one, it's not properly integrated with GNU Readline. There's an
`.echo` meta-command that turns command echoing on and off. That is,
it repeats your command back to you. Useful in some circumstances,
though not mine. The bug is that this echo is separate from GNU
Readline's echo. When Readline is active and `.echo` is enabled, there
are actually *two* echos. Turn it off and there's one echo.

##### Pseudo-terminals

Under some circumstances, like when communicating over a pipe rather
than a PTY, Readline will mostly become deactivated. This would have
been a workaround, but when Readline is disabled sqlite3 heavily
buffers its output. This breaks any sort of interaction. Even worse,
on Windows [stderr is not always unbuffered][buffer], so sqlite3's
error messages may not appear for a long time (another bug).

Besides the problem of getting Readline to shut up, another problem is
getting Readline to stop acting on control characters. The first 32
characters in ASCII are control characters. A pseudo-terminal (PTY)
that is not in raw mode will immediately act upon any control
characters it sees. There's no escaping them.

Emacs communicates with subprocesses through a PTY by default
(probably an early design mistake), limiting the kind of data that can
be transmitted. You can try this yourself in a comint mode sometime
where a subprocess is used (not a socket like SLIME). Fire up `M-x
sql-sqlite` (part of Emacs) and try sending a string containing byte
0x1C (28, file separator). You can type one by pressing `C-q C-\`.
Send that byte and the subprocess dies.

There are two ways to work around this. One is to use a pipe (bind
`process-connection-type` to nil). Pipes don't respond to control
characters. This doesn't work with sqlite3 because of the
previously-mentioned buffering issue.

The other way to work around this is to put the PTY in raw mode.
Unfortunately there's no function to do this so you need to call
`stty`. Of course, this program needs to run on the same PTY, so a
`start-process-shell-command` is required.

~~~cl
(start-process-shell-command name buffer "stty raw && <your command>")
~~~

Windows has neither `stty` nor PTYs (nor any of PTY's issues) so
you'll need to check the operating system before starting the process.
Even this still doesn't work for sqlite3 because Readline itself will
respond to control characters. There's no option to disable this.

There's a package called [esqlite][esqlite] that is also a SQLite
front-end. It's built to use sqlite3 and therefore suffers from all of
these problems.

#### A Custom SQLite Binary

Since sqlite3 proved unreliable I developed my own protocol and
external program. It's just a tiny bit of C that accepts a SQL string
and returns results as an s-expression. I'm not longer constrained to
storing readable values, but I'm still keeping that paradigm. First,
it keeps the C glue program simple and, more importantly, I can rely
entirely on the Emacs reader to parse the results. This makes
communication between Emacs and the subprocess as fast as it can
possibly be. The reader is faster than any possible Elisp program.

As I mentioned before, this C program is compiled when possible, and
otherwise a pre-built binary is fetched from my server (popular
platforms only, obviously). It's likely EmacSQL will have at least one
working back-end on whatever you're using.

### Other Back-ends

Both PostgreSQL and MySQL are also supported, though these require the
user have the appropriate client programs installed (psql or mysql).
Both of these are much better behaved than sqlite3 and, with the
`stty` trick, each can reliably be used without any special help. Both
pass all of the unit tests, so, in theory, they'll work just as well
as SQLite.

To use them with the example at the beginning of this article, require
`emacsql-psql` or `emacsql-mysql`, then swap `emacsql-connect` for the
constructors `emacsql-psql` or `emacsql-mysql` (along with the proper
arguments). All three of these constructors return an
`emacsql-connection` object that works with the same API.

EmacSQL only goes so far to normalize the interfaces to these
databases, so for any non-trivial program you may not be able to swap
back-ends without some work. All of the EmacSQL functions that operate
on connections are generic functions (EIEIO), so changing back-ends
will only have an effect on the program's SQL statements. For example,
if you use q SQLite-ism (dynamic typing) it won't translate to either
of the other databases should they be swapped in.

I'll cover the connections API, and what it takes to implement a new
back-end, in a future post. Outside of the PTY caveats, it's actually
very easy. The MySQL implementation is just 80 lines of code.

### EmacSQL's Future

I hope this becomes a reliable and trusted database solution that
other packages can depend upon. Twice so far, the pastebin demo and
Elfeed, I've really wanted something like this and, instead, ended up
having to hack together my own database.

I've already started a branch on Elfeed re-implementing its database
in EmacSQL. Someday it may become Elfeed's primary database if I feel
there's no disadvantage to it. EmacSQL builds SQLite with the
full-text search engine enabled, which opens to the door to a
powerful, fast Elfeed search API. Currently the main obstacle is
actually Elfeed's database API being somewhat incompatible with ACID
database transactions — shortsightedness on my part!


[emacsql]: https://github.com/skeeto/emacsql
[pastebin]: /blog/2012/12/29/
[readable]: /blog/2013/12/30/#almost_everything_prints_readably
[finalize]: /blog/2014/01/27/
[elfeed]: /blog/2013/09/09/
[buffer]: http://sqlite.1065341.n5.nabble.com/Command-line-shell-not-flushing-stderr-when-interactive-td73340.html
[esqlite]: https://github.com/mhayashi1120/Emacs-esqlite
[melpa]: http://melpa.milkbox.net/#/emacsql
[skewer]: /blog/2012/10/31/
[book]: http://www.amazon.com/gp/product/0596521189/ref=as_li_qf_sp_asin_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0596521189&linkCode=as2&tag=nullprogram-20
