---
title: 'Debugging Emacs or: How I Learned to Stop Worrying and Love DTrace'
layout: post
date: 2018-01-17T23:59:49Z
tags: [emacs, elfeed, bsd]
uuid: a55cabc9-2d87-30a4-9066-9ec5e45b8bce
excerpt_separator: <!--more-->
---

*Update: This article was featured on [BSD Now 233][feature] (starting
at 21:38).*

For some time [Elfeed][elfeed] was experiencing a strange, spurious
failure. Every so often users were [seeing an error][bug] (spoiler
warning) when updating feeds: "error in process sentinel: Search
failed." If you use Elfeed, you might have even seen this yourself.
From the surface it appeared that curl, tasked with the
[responsibility for downloading feed data][curl], was producing
incomplete output despite reporting a successful run. Since the run
was successful, Elfeed assumed certain data was in curl's output
buffer, but, since it wasn't, it failed hard.

<!--more-->

Unfortunately this issue was not reproducible. Manually running curl
outside of Emacs never revealed any issues. Asking Elfeed to retry
fetching the feeds would work fine. The issue would only randomly rear
its head when Elfeed was fetching many feeds in parallel, under
stress. By the time the error was discovered, the curl process had
exited and vital debugging information was lost. Considering that
this was likely to be a bug in Emacs itself, there really wasn't a
reliable way to capture the necessary debugging information from
within Emacs Lisp. And, indeed, this later proved to be the case.

A quick-and-dirty work around is to use `condition-case` to catch and
swallow the error. When the bizarre issue shows up, rather than fail
badly in front of the user, Elfeed could attempt to swallow the error
— assuming it can be reliably detected — and treat the fetch as simply
a failure. That didn't sit comfortably with me. Elfeed had done its
due diligence checking for errors already. *Someone* was lying to
Elfeed, and I intended to catch them with their pants on fire.
Someday.

I'd just need to witness the bug on one of my own machines. Elfeed is
part of my daily routine, so surely I'd have to experience this issue
myself someday. My plan was, should that day come, to run a modified
Elfeed, instrumented to capture extra data. I would have also routinely
run Emacs under GDB so that I could inspect the failure more deeply.

For now I just had to wait to [hunt that zebra][zebra].

### Bryan Cantrill, DTrace, and FreeBSD

Over the holidays I re-discovered [Bryan Cantrill][bc], a systems
software engineer who worked for Sun between 1996 and 2010, and is most
well known for [DTrace][dtrace]. My first exposure to him was in a [BSD
Now interview][bsd] in 2015. I had re-watched that interview and decided
there was a lot more I had to learn from him. He's become a personal
hero to me. So I scoured the internet for [more of his writing and
talks][binge]. Besides what I've already linked in this article, here
are a couple more great presentations:

* [Oral Tradition in Software Engineering][oral]
* [Fork Yeah! The Rise and Development of illumos][fork]

You can also find some of his writing [scattered around the DTrace
blog][bmc].

Some interesting operating system technology came out of Sun during
its final 15 or so years — most notably DTrace and ZFS — and Bryan
speaks about it passionately. Almost as a matter of luck, most of it
survived the Oracle acquisition thanks to Sun releasing it as open
source in just the nick of time. Otherwise it would have been lost
forever. The scattered ex-Sun employees, still passionate about their
prior work at Sun, along with some of their old customers have since
picked up the pieces and kept going as a community under the name
[illumos][illumos]. It's like an open source flotilla.

Naturally I wanted to get my hands on this stuff to try it out for
myself. Is it really as good as they say? Normally I stick to Linux,
but it (generally) doesn't have these Sun technologies. The main
reason is license incompatibility. Sun released its code under the
[CDDL][cddl], which is incompatible with the GPL. Ubuntu *does*
[infamously include ZFS][ubuntu], but other distributions are
unwilling to take that risk. Porting DTrace is a serious undertaking
since it's got its fingers throughout the kernel, which also makes the
licensing issues even more complicated.

(*Update Feburary 2018*: [DTrace has been released under the
GPLv2][news], allowing it to be legally integrated with Linux.)

Linux has a reputation for Not Invented Here (NIH) syndrome, and these
licensing issues certainly contribute to that. Rather than adopt ZFS
and DTrace, they've been reinvented from scratch: btrfs instead of
ZFS, and [a slew of partial options][trace] instead of DTrace.
Normally I'm most interested in system call tracing, and my go to is
[strace][strace], though it certainly has its limitations — including
this situation of debugging curl under Emacs. Another famous example
of NIH is Linux's [`epoll(2)`][epoll], which is a [broken][epoll1]
[version][epoll2] of BSD [`kqueue(2)`][kqueue].

So, if I want to try these for myself, I'll need to install a
different operating system. I've dabbled with [OmniOS][omnios], an OS
built on illumos, in virtual machines, using it as an alien
environment to test some of my software (e.g. [enchive][enchive]).
OmniOS has a philosophy called [Keep Your Software To Yourself][kysty]
(KYSTY), which is really just code for "we don't do packaging."
Honestly, you can't blame them since [they're a tiny community][cs].
The best solution to this is probably [pkgsrc][pkgsrc], which is
essentially a universal packaging system. Otherwise [you're on your
own][home].

There's also [openindiana][oi], which is a more friendly
desktop-oriented illumos distribution. Still, the short of it is that
you're very much on your own when things don't work. The situation is
like running Linux a couple decades ago, when it was still difficult
to do.

If you're interested in trying DTrace, the easiest option these days is
probably [FreeBSD][freebsd]. It's got a big, active community, thorough
documentation, and a huge selection of packages. Its license (the *BSD
license*, duh) is compatible with the CDDL, so both ZFS and DTrace have
been ported to FreeBSD.

### What is DTrace?

I've done all this talking but haven't yet described what [DTrace
really is][tut]. I won't pretend to write my own tutorial, but I'll
provide enough information to follow along. DTrace is a tracing
framework for debugging production systems *in real time*, both for
the kernel and for applications. The "production systems" part means
it's stable and safe — using DTrace won't put your system at risk of
crashing or damaging data. The "real time" part means it has little
impact on performance. You can use DTrace on live, active systems with
little impact. Both of these core design principles are vital for
troubleshooting those really tricky bugs that only show up in
production.

There are DTrace *probes* scattered all throughout the system: on
system calls, scheduler events, networking events, process events,
signals, virtual memory events, etc. Using a specialized language
called D (unrelated to the general purpose programming language D),
you can dynamically add behavior at these instrumentation points.
Generally the behavior is to capture information, but it can also
manipulate the event being traced.

Each probe is fully identified by a 4-tuple delimited by colons:
provider, module, function, and probe name. An empty element denotes a
sort of wildcard. For example, `syscall::open:entry` is a probe at the
beginning (i.e. "entry") of `open(2)`. `syscall:::entry` matches all
system call entry probes.

Unlike strace on Linux which monitors a specific process, DTrace
applies to the entire system when active. To run curl under strace
from Emacs, I'd have to modify Emacs' behavior to do so. With DTrace I
can instrument every curl process without making a single change to
Emacs, and with negligible impact to Emacs. That's a big deal.

So, when it comes to this Elfeed issue, FreeBSD is much better poised
for debugging the problem. All I have to do is catch it in the act.
However, it's been months since that bug report and I'm not really
making this connection yet. I'm just hoping I eventually find an
interesting problem where I can apply DTrace.

### FreeBSD on a Raspberry Pi 2

So I've settled in FreeBSD as the playground for these technologies, I
just have to decide where. I could always run it in a virtual machine,
but it's always more interesting to try things out on real hardware.
[FreeBSD supports the Raspberry Pi 2][raspi2] as a Tier 2 system, and
I had a Raspberry Pi 2 sitting around collecting dust, so I put it to
use.

I wrote the image to an SD card, and for a few days I stretched my
legs on this new system. I cloned a couple dozen of my own git
repositories, ran the builds and the tests, and just got a feel for
things. I tried out the ports system for the first time, mainly to
discover that the low-powered Raspberry Pi 2 takes days to build some
of the packages I want to try.

I [mostly program in Vim these days][vim], so it's some days before I
even set up Emacs. Eventually I do build Emacs, clone my
configuration, fire it up, and give Elfeed a spin.

And that's when the "search failed" bug strikes! Not just once, but
dozens of times. Perfect! This low-powered platform is the jackpot for
this particular bug, triggering it left and right. Given that I've got
DTrace at my disposal, it's *the* perfect place to debug this.
Something is lying to Elfeed and DTrace will play the judge.

Before I dive in I see three possibilities:

1. curl is reporting success but truncating its output.
2. Emacs is quietly truncating curl's output.
3. Emacs is misinterpreting curl's exit status.

With Dtrace I can observe what every curl process writes to Emacs, and
I can also double check curl's exit status. I come up with the
following (newbie) DTrace script:

~~~
syscall::write:entry
/execname == "curl"/
{
    printf("%d WRITE %d \"%s\"\n",
           pid, arg2, stringof(copyin(arg1, arg2)));
}

syscall::exit:entry
/execname == "curl"/
{
    printf("%d EXIT  %d\n", pid, arg0);
}
~~~

The `/execname == "curl"/` is a predicate that (obviously) causes the
behavior to only fire for curl processes. The first probe has DTrace
print a line for every `write(2)` from curl. `arg0`, `arg1`, and
`arg2` correspond to the arguments of `write(2)`: fd, buf, count. It
logs the process ID (pid) of the write, the length of the write, and
the actual contents written. Remember that these curl processes are
run in parallel by Emacs, so the pid allows me to associate the
separate writes and the exit status.

The second probe prints the pid and the exit status (the first argument
to `exit(2)`).

I also want to compare this to exactly what is delivered to Elfeed when
curl exits, so I modify the [process sentinel][sentinel] — the callback
that handles a subprocess exiting — to call `write-file` before any
action is taken. I can compare these buffer dumps to the logs produced
by DTrace.

There are two important findings.

First, when the "search failed" bug occurs, the buffer was completely
empty (95% of the time) or truncated at the end of the HTTP headers
(5% of the time), right at the blank line. DTrace indicates that curl
did its job to the full, so it's Emacs who's the liar. It's not
delivering all of curl's data to Elfeed. That's pretty annoying.

Second, **curl was line-buffered**. Each line was a separate,
independent `write(2)`. I was certainly *not* expecting this. Normally
the C library only does line buffering when the output is a terminal.
That's because it's guessing a user may be watching, expecting the
output to arrive a line at a time.

Here's a sample of what it looked like in the log:

~~~
88188 WRITE 32 "Server: Apache/2.4.18 (Ubuntu)
"
88188 WRITE 46 "Location: https://blog.plover.com/index.atom
"
88188 WRITE 21 "Content-Length: 299
"
88188 WRITE 45 "Content-Type: text/html; charset=iso-8859-1
"
88188 WRITE 2 "
"
~~~

Why would curl think Emacs is a terminal?

*Oh.* That's right. *This is the [same problem I ran into four years
ago when writing EmacSQL][emacsql].* By default Emacs connects to
subprocesses through a pseudo-terminal (pty). I called this a mistake
in Emacs back then, and I still stand by that claim. The pty causes
weird, annoying problems for little benefit:

* Interpreting control characters. Hope you weren't transferring binary
  data!
* Subprocesses will generally get line buffered. This makes them
  slower, though in some situations it might be desirable.
* Stdout and stderr get mixed together. (Optional since Emacs 25.)
* *New!* There's a bug somewhere in Emacs that causes truncation when
  ptys are used heavily in parallel.

Just from eyeballing the DTrace log I knew what to do: dump the pty
and switch to a pipe. This is controlled with the
`process-connection-type` variable, and fixing it [is a
one-liner][fix].

Not only did this completely resolve the truncation issue, Elfeed is
noticeably faster at fetching feeds on all machines. It's no longer
receiving mountains of XML one line at a time, like sucking pudding
through a straw. It's now quite zippy even on my Raspberry Pi 2, which
had *never* been the case before (without the "search failed" bug).
Even if you were never affected by this bug, you will benefit from the
fix.

I haven't officially reported this as an Emacs bug yet because
reproducibility is still an issue. It needs something better than
"fire off a bunch of HTTP requests across the internet in parallel
from a Raspberry Pi."

The fix reminds me of that [old boilermaker story][hammer] about
charging a lot of money just to swing a hammer. Once the problem
arose, **DTrace quickly helped to identify the place to hit Emacs with
the hammer**.

*Finally, a big thanks to alphapapa for originally taking the time to
report this bug months ago.*


[bc]: https://en.wikipedia.org/wiki/Bryan_Cantrill
[binge]: http://dtrace.org/blogs/bmc/2018/02/03/talks/
[bmc]: http://dtrace.org/blogs/bmc/
[bsd]: https://www.youtube.com/watch?v=l6XQUciI-Sc
[bug]: https://github.com/skeeto/elfeed/issues/248
[cddl]: https://opensource.org/licenses/CDDL-1.0
[cs]: https://utcc.utoronto.ca/~cks/space/blog/solaris/IllumosSupportLimits
[curl]: /blog/2016/06/16/
[dtrace]: http://dtrace.org/blogs/about/
[elfeed]: https://github.com/skeeto/elfeed
[emacsql]: /blog/2014/02/06/
[enchive]: /blog/2017/03/12/
[epoll]: http://man7.org/linux/man-pages/man7/epoll.7.html
[epoll1]: https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/
[epoll2]: https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/
[feature]: https://www.youtube.com/watch?v=Xi_pX2QIzho
[fix]: https://github.com/skeeto/elfeed/commit/945765a57d2f27996b6a43bc62e803dc167d1547
[fork]: https://www.youtube.com/watch?v=-zRN7XLCRhc
[freebsd]: https://www.freebsd.org/
[hammer]: https://www.buzzmaven.com/old-engineer-hammer-2/
[home]: /blog/2017/06/19/
[illumos]: https://illumos.org/
[kqueue]: https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2
[kysty]: https://omnios.omniti.com/wiki.php/KYSTY
[news]: https://gnu.wildebeest.org/blog/mjw/2018/02/14/dtrace-for-linux-oracle-does-the-right-thing/
[oi]: https://www.openindiana.org/
[omnios]: https://omnios.omniti.com/
[oral]: https://www.youtube.com/watch?v=4PaWFYm0kEw
[pkgsrc]: https://www.pkgsrc.org/
[raspi2]: https://wiki.freebsd.org/FreeBSD/arm/Raspberry%20Pi
[sentinel]: http://www.gnu.org/software/emacs/manual/html_node/elisp/Sentinels.html
[strace]: https://en.wikipedia.org/wiki/Strace
[trace]: http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html
[tut]: https://wiki.freebsd.org/DTrace/Tutorial
[ubuntu]: https://insights.ubuntu.com/2016/02/18/zfs-licensing-and-linux/
[vim]: /blog/2017/04/01/
[zebra]: https://www.youtube.com/watch?v=fE2KDzZaxvE
