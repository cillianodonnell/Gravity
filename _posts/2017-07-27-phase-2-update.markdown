---                                                                             
layout: post                                                                    
title:  "Phase 2 Update"                                    
date:   2017-07-26 13:41:00                                              
categories: rtems, gsoc                                                     
--- 
I waited for this update as the first half of the month just consisted of trying
to get patches merged into upstream QEMU and then re-orienting myself with the
main project of getting the coverage tools working. Then I got stuck and then I 
was on holidays for a week, so it didn't seem like there was much to write 
about. Nonetheless I want to give an quick overview about what went on.

It started just going back and forth in emails with Frederic Konrad from the 
Couverture-Qemu project and doing a bit of testing. I struggled to fix the
compile issues and Frederic solved them in a couple of hours what I couldn't do
for days. Quite impressive and it just goes to show how far I have to go to 
become a professional developer. It then seemed like I needed to hear back from
the original patch creator just to confirm everything was as he intended now
that it was rebased against current qemu master, so this was set aside for a 
while.

I got back to my RTEMS Tester work, initially it couldn't find the symbol sets
to run coverage on and I came up with elaborate reasons for why that was before
realising the path in the config file was just wrong (I had convinced myself it
was right... I need to be more careful). 

It was running now, which was great to see after months of being broken.
However there was some issues, it would finish with 'Trace Block is Inconsistent
with Coverage Map..' messages, there was missing Gcov files and on occasion it
would lock up in the middle of its coverage analysis runs and just hang idle
indefinitely.

I spent a lot of time with the trace block inconsistent with coverage map
problem and learned to use GDB in a one to one with my mentor Joel to figure it
out. After a few late nights, I emailed the devel list with my understanding so far..

{% highlight bash %}
*** Trace block is inconsistent with coverage map
*** Trace block (0x4000c2cc - 0x4000c2ec) for 36 bytes
*** Coverage map /home/cpod/coverage test/leon3/coverage/base sp.exe.cov

  -----------------------------

The coverage map is 24 bytes:

(gdb) p/x entry
$2 = {pc = 0x4000c2cc, size = 0x24, op = 0x12}

  -----------------------------

The disassembly block in question is:

4000c2cc < Objects Get information id>:
4000c2cc:   83 32 20 18     srl  %o0, 0x18, %g1

Objects Information * Objects Get information id(
  Objects Id  id
)
{
  return  Objects Get information(
4000c2d0:   93 32 20 1b     srl  %o0, 0x1b, %o1
4000c2d4:   90 08 60 07     and  %g1, 7, %o0
4000c2d8:   82 13 c0 00     mov  %o7, %g1
4000c2dc:   40 00 00 02     call  4000c2e4 < Objects Get information>
4000c2e0:   9e 10 40 00     mov  %g1, %o7

4000c2e4 < Objects Get information>:

Objects Information * Objects Get information(
  Objects APIs   the api,
  uint16 t       the class
)
{
4000c2e4:   9d e3 bf a0     save  %sp, -96, %sp
  Objects Information info;
  int the class api maximum;

  if ( !the class )
4000c2e8:   80 a6 60 00     cmp  %i1, 0
4000c2ec:   02 80 00 19     be  4000c350 < Objects Get information+0x6c>
4000c2f0:   01 00 00 00     nop

  ------------------------------------

I have checked this out on base sp.exe and ticker.exe where the same
inconsistency appears in both. Couverture-QEMU decides the trace block
encompasses that entire block

Whereas from the covoar side the objdump is always processed line by
line and a regex is checked to determine what kind of line it is.

408       items = sscanf(
409         inputBuffer,
410         "%x <%[^>]>%c",
411         &offset, symbol, &terminator1
412       );

If there is a match for all 3 items then this is a new function

415       if ((items == 3) && (terminator1 == ':')) {
416
417         endAddress = executableInformation->getLoadAddress() + offset - 1;


So the objdump above is always processed as 2 seperate sections.

One for:
4000c2cc < Objects Get information id>:


And one for:
4000c2e4 < Objects Get information>:

The size from Objects Get information id to Objects Get information is
24 bytes which is the coverage map.
The size of both functions combined is 36 bytes which is the
Couverture trace block.

Couverture could possibly be doing something more sophisticated to
determine the end of the block or it could just be wrong. So the next
thing I'm doing is figuring out exactly how Couverture is processing
this.
{% endhighlight %}

After a bit of back and forth email with Joel, we decided this was too 
restrictive a check and not really neccessary. This section was removed and I
checked if we were getting sensible results. Sure enough the section marked
not taken in the coverage map was the same as the branch marked '0x12' by
couverture-qemu which is a bitmap detailing 'branch fully exectuted' and 'branch
not taken'. The taken branches are op code 0x11.

This turned out to be a simple solution but it took quite a while to understand
the problem so a decision could be made. That has been a recurring theme so far,
I've spent most of my time just understanding what other people have written and
how things work. The writing of my own code has been minimal, as compared to a 
university assignment which is nothing but coding my own solution from scratch 
every time.

Now as the project has passed through 2 sets of hands before me, there is an 
urgency for me to finally finish the job and get this merged. I've begun this
process, the --coverage option is implemented in the wrong place, the error 
class is not being used to report errors, missing license, unit tests still in
files, camelCase being used instead of underscores and general style issues not
matching with what is around it.

I rushed to fix all of these on my main branch and ending up breaking what I had
and had to start again. Surprisingly I was able to get back to where I was in 
about a day, which proves that I've come along way since the beginning of GSOC.
Lesson to be learned about git workflow is never break what's already working, 
create a new branch for each small change, test it and then merge it if its 
working. Repeat for the next problem. Anyway on the plus side the lock-up 
happens less frequently and the output was cleared up a bit, so I ended up with
an improved version.

The main thing left to fix is the covoar lock-up problem and there could be some
other merging problems after a second review so I'll have to see. It looks like
this phase 2 work will spill into phase 3 and I'm not sure what will happen to
the original plans of refactoring covoar to produce XML. It's hard to say if 
there will be time anymore. However it goes, I will definitely get the work 
merged and working to a standard that everyone is happy with. Then I could work
on the XML output in my spare time post GSOC.
