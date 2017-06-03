---
layout: post
title:  "Reviving The Old RTEMS Tester Work"
date:   2017-06-03 19:50:31 +0530
categories: rtems
---
The first thing I did is learn how to use git. The workflow I settled on is keep a pristine RTEMS master and make a branch
to work on with 'git checkout -b branchname'. Then try to gather commits into logically separate  ideas, problems. Don't commit
too early as it can tricky to change them after the fact, I still haven't figured it out. Then 'git push to_github' and 
'git format-patch origin' and 'git send-email' to devel@rtems.org for review and hopefully merging. I also brushed up on
my Python skills by working through [google-python-tutorial].

There had been 2 previous student who had worked on this project before for Summer of Code in Space (like GSOC but run by the
European Space Agency), their work is [SOCIS-2014] and [SOCIS-2015]. They both left a series of patches, some of which worked
as they were adding new files and some of which no longer fitted where it was supposed to as there has been much development
that has taken place since the 2015 work. I used 'git apply --reject patches/' on each set and would have a log of x applied,
x partially applied with x rejects and there would be .rej files scattered around the directories that I could sort through
and make a judgment call on what should go where. Alot of them were just out of place, the code now fit too far away for
git to guess where it should be. There were other cases were it was neccessary to judge if something had been updated and 
now supercedes that old decision and then a few small fixes to be added of my own.

My notes on the 2014 work are scant as I was going through these while still busy with college work, but from the few lines I
recorded the qemu configuration file qemu.cfg did not take at all and there might be a doubling up of bld.program in covoar 
wscript, which I'm not quite sure what I meant by that anymore. Anyway my notes from here onwards are better.

When I finished going through the 2014 patches, there was still nothing building or working, so I moved onto the 2015 patches
as their work built on each others and the 2015 stuff overwrote some of the 2014 work. As before I 'git apply --reject patches/'
and ended up with 14 applied cleanly and another 8 files with with 23 rejected hunks between them. I 'git pull origin' and see
that there's a conflict with options.py that I'll have to fix later when I commit my changes and update my master branch and
'git rebase'.

The first thing I was wondering about is in covoar/wscript, which is like a Makefile but for the waf build sytem (written in 
Python). Even though both 2014 and 2015 work uses rtl-host which is a run-time linking mechanism that is added to the top of
rtems-tools, the 2014 work uses the rld....h header files in rtl-host and the 2015 work uses them from rtemstoolkit. It's the 
exact same files in both but I couldn't seem to figure out if it made a difference, I went with the 2015 and it is the most 
recent, both seemed to work.

At this stage I didn't understand things too well so when presented with two choices from each work I might just pick one
and see how it went. For instance in rt/test.py:

{% highlight bash %}
coverage_enabled = opts.opts['coverage']
# or I could use
coverage_enabled = opts.coverage()
{% endhighlight %}

I was finding it difficult to track down exactly what opts was generating, there is and opts in all the .py files as a standard
naming convention for the options loaded in. This opts is from options.py which backtracks to anothers options file when I followed that trail and I just couldn't get a clear picture of what was in there to make a decision (hadn't figured out I 
could just print this at runtime, although there was nothing to run yet so maybe that's why).

Or again from tester/rtems/testing/gdb.cfg:

{% highlight bash %}
%if %{_coverage}
# or I could use
%if %{defined _coverage}
{% endhighlight %}

I used the first one and it later turned into a runtime error on rtems-test (RTEMS Tester cmd) and was fixed using the second.

Similar choices in qemu.cfg and I have yet to decide which options would be best in there.

{% highlight bash %}
# So far all 3 options leave runtime errors and I suspect it is the bsp qemu 
# options causing the problems.
#%define qemu_opts_base   -no-reboot -monitor none -serial stdio -nographic     
#%define qemu_opts_base   -no-reboot -monitor null -serial stdio -nographic     
%define qemu_opts_base   -no-reboot -serial null -serial mon:stdio -nographic   
%define qemu_opts_no_net -net none
{% endhighlight %}

After sorting through all the rejected patches and making my decisions, it was time for a first build of rtems-tools with waf.
'./waf configure' went fine and './waf build install' turned up a misplaced macro 'source not found: 'RLD' in bld' in 
covoar/wscript. The answer to this was in the original patches and it was just misplaced when added by hand, still took a while
to track that down though.

The next problem would result in my first real fix, another build error 

{% highlight %}
rld-process.h:175:36: error: ‘strings’ in namespace ‘rld’ does not name a type
       void write_lines (const rld::strings& ss);
{% endhighlight %}

I had originally thought this was referring to the return type and I was changing const to void or removing it and none of that
worked and after some advice from my mentor Joel on how to think about this so I could solve it myself, I realised it couldn't 
find the definition of 'strings'. I detoured a bit trying to understand the waf build system as I thought the problem was in
covoar/wscript and some linking mechanism there before solving it by adding '#include "rld.h"' to rld-process.h ( I had tried
#include <rld.h> earlier, not realising this was incorrect syntax and then went on the wscript detour). This fix in
rtemstoolkit/rld-process.h was submitted to rtems devel list and merged. After this the build was successful.

Now it's time for a first run through the RTEMS Tester framework, the following command runs the testsuite for PC386 BSP
with coverage enabled with the --coverage flag.

{% highlight %}
$HOME/development/rtems/test/rtems-tools/tester/rtems-test --rtems-bsp=pc386 --coverage --log=log-pc386.log --rtems-tools=$HOME/development/rtems/4.12 $HOME/development/rtems/pc386/i386-rtems4.12/c/pc386/testsuites
{% endhighlight %}

There was an initial 'SyntaxError: Non-ASCII character '\xc4' in .../coverage.py' caused by a strange character from a Polish
name of the 2014 student. Also rtems-test was originally looking for the executables at '...pc386/testsuites' as the previous
student had listed but they are actually found at the longer path above '...pc386/i386-rtems4.12/c/pc386/testsuites'. These were
quickly fixed.

The tests actually ran now. All 446 were invalid and defaulted to dry-run due to errors in qemu.cfg but it was nice to see a
first run through nonetheless. The error consisted of the qemu command and all its options

{% highlight %}
qemu.cfg:81: execute failed: qemu-system-i386 -m 128 -boot b... the rest of the options
common and bsp specific ... tmtimer01.exe.cov: exit-code:2
{% endhighlight %}

This exit code meant something like directory or path not found, when the coverage flag was removed it had exit code: 1
which is more of a general error but based on the context seems to amount to about the same thing here. By chance I moved
the coverage flag to last place, just before the executable and strangely the qemu.cfg errors disappeared and the tests
all took some time to run (12 min for 13 sample tests vs 28s for 446 tests) and all timed out now.

However I noticed that coverage analysis tried to run whether the --coverage flag had been added or not, Leon 3 BSP showed
this output without --coverage.

{% highlight %}
RTEMS Testing - Tester, 4.12 (b047c7737e9d modified)
Coverage analysis requested
Traceback (most recent call last):
...
File "/home/cpod/development/rtems/test/rtems-tools/tester/rt/test.py", line 300, in run
    coverage = coverage.coverage_run(opts.defaults)
File "/home/cpod/development/rtems/test/rtems-tools/tester/rt/coverage.py", line 285, in __init__
    self.config_map = self.macros.macros['coverage']
KeyError: 'coverage'
{% endhighlight %}

From this I figured that with --coverage removed test.py shouldn't call coverage.py at all. The problem is that there was
an if statement that always seemed to be true, which was checked with a print statement.

{% highlight %}
coverage_enabled = opts.opts['coverage']
if coverage_enabled:
            print('\nThis path is taken\n') # added by me to check if it always ran
            import coverage               
            from rtemstoolkit import check           
            log.notice("Coverage analysis requested")       
            opts.defaults.load('%%{_configdir}/coverage.mc')              
            if not check.check_exe('__covoar', opts.defaults['__covoar']):
                 raise error.general("Covoar not found!")   
            coverage = coverage.coverage_run(opts.defaults)
            coverage.prepareEnvironment()    
{% endhighlight %}

So I checked coverage_enabled and it was printing 1 for --coverage and 0 wothout --coverage. Which should be a proxy for
true and false and if coverage_enabled should check automatically check for true, I'm not sure why it didn't but I solved it
by explicitly asking it to check 'if coverage_enabled == True:' This solution worked again for the next traceback
'UnboundLocalError: local variable 'coverage' referenced before assignment' another if statement was always true.

Now with no --coverage the tests actually have what looks to be a reasonable set of results and the current RTEMS Tester has 
been insulated from the new coverage work, so at least it isn't breaking anything that is already there. This is true for 
Leon 2 and Leon 3. PC386 is only supported by the new coverage work so it isn't working yet.

{% highlight %}
RTEMS Testing - Tester, 4.12 (b047c7737e9d modified)
[ 3/13] p:0  f:0  u:0  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: cdtest.exe
[ 2/13] p:0  f:0  u:0  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: capture.exe
[ 1/13] p:0  f:0  u:0  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: base_sp.exe
[ 4/13] p:0  f:0  u:0  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: fileio.exe
[ 5/13] p:1  f:1  u:1  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: hello.exe
[ 6/13] p:1  f:1  u:1  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: cxx_iostream.exe
[ 7/13] p:1  f:1  u:1  e:0  I:0  B:0  t:0  i:0  | sparc/leon3: loopback.exe
[ 8/13] p:3  f:1  u:1  e:0  I:0  B:0  t:0  i:1  | sparc/leon3: minimum.exe
[10/13] p:3  f:1  u:1  e:0  I:0  B:0  t:0  i:1  | sparc/leon3: paranoia.exe
[ 9/13] p:3  f:1  u:1  e:0  I:0  B:0  t:0  i:1  | sparc/leon3: nsecs.exe
[11/13] p:3  f:1  u:1  e:0  I:0  B:0  t:0  i:2  | sparc/leon3: pppd.exe
[12/13] p:3  f:1  u:1  e:0  I:0  B:0  t:1  i:2  | sparc/leon3: ticker.exe
[13/13] p:4  f:1  u:1  e:0  I:0  B:0  t:1  i:2  | sparc/leon3: unlimited.exe
Passed:         5
Failed:         1
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        4
Invalid:        2
Total:         13
Average test time: 0:00:28.078878
Testing time     : 0:06:05.025426
{% endhighlight %}

With --coverage there is gdb.cfg errors as it turns out the coverage flag is not triggering the coverage.mc files to be used
and changing the bsp option rtems-bsp=leon3-coverage triggers the leon3-coverage.mc file and now Leon 2, Leon 3 and PC386
all run with the qemu.cfg errors back, which I'm suspecting is something to do the qemu options needing to be changed.
This is the current state of the project.

I know that some of these problems might seem trivial to some people reading this and I think that some of them were quite 
simple when I look back on them too. However as a novice they took me quite some time to figure out exactly what was
happening and to find these simple answers. So I'm recording the development process as it was and not how I would of liked
it to have been.

[google-python-tutorial]: https://developers.google.com/edu/python/
[SOCIS-2014]: http://kmiesowicz.blogspot.ie/p/esa-socis-2014.html
[SOCIS-2015]: http://socis2015rtems.blogspot.ie/
