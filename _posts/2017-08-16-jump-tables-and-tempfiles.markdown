---
layout: post
title:  "Jump Tables a Recurring Problem."
date:   2017-08-16 19:04:00
categories: rtems, gsoc
---
Another long delay in updates as I've spent so much time on fixing one simple
problem. The problem of jump tables being added to the end of objdump files.
There was a lot of detective work in gdb that I don't have anything tangible to
show for the time spent. The amount of code needed to solve the problem was
minimal. As with all the other phases of the project I've spent most of my time
figuring out how the pieces of the puzzle fit together and reading other peoples
code. This may be just the nature of large coding projects, as opposed to small
college projects where a lot of code is written and there is no interaction
with existing code.

INFO messages, that were once error messages, would appear showing size
mismatches in the same symbols from different exe's.

{% highlight bash %}
INFO: DesiredSymbols::createCoverageMap - Attempt to create unified
coverage maps for TOD_TICKS_PER_SECOND_method with different sizes
(/home/cpod/coverage_test/leon3/coverage/fsrfsbitmap01.exe/40 !=
/home/cpod/coverage_test/leon3/coverage/block01.exe/92)
INFO: DesiredSymbols::createCoverageMap - Attempt to create unified
coverage maps for TOD_TICKS_PER_SECOND_method with different sizes
(/home/cpod/coverage_test/leon3/coverage/fsrofs01.exe/92 !=
/home/cpod/coverage_test/leon3/coverage/block01.exe/40)
INFO: DesiredSymbols::createCoverageMap - Attempt to create unified
coverage maps for _Thread_queue_Extract_with_proxy with different
sizes (/home/cpod/coverage_test/leon3/coverage/ftp01.exe/12 !=
/home/cpod/coverage_test/leon3/coverage/base_sp.exe/248)
INFO: DesiredSymbols::createCoverageMap - Attempt to create unified
coverage maps for _Thread_queue_Extract_with_proxy with different
sizes (/home/cpod/coverage_test/leon3/coverage/gxx01.exe/248 !=
/home/cpod/coverage_test/leon3/coverage/base_sp.exe/12)
INFO: DesiredSymbols::createCoverageMap - Attempt to create unified
coverage maps for _Heap_Walk with different sizes
(/home/cpod/coverage_test/leon3/coverage/heapwalk.exe/1656 !=
/home/cpod/coverage_test/leon3/coverage/fileio.exe/1600)
...
{% endhighlight %}

They were mostly about block01.exe and base_sp.exe being matched with other
exe's. So I would pair those with other exe's and try and produce just just one
instance of the INFO message and then examine the objdumps and compare the
symbols with the discrepancies.

{% highlight bash %}
block01.exe is 40 bytes

36363 4001659c <TOD_TICKS_PER_SECOND_method>:
36364
36365 uint32_t TOD_TICKS_PER_SECOND_method(void)
36366 {
36367   return (TOD_MICROSECONDS_PER_SECOND /
36368       rtems_configuration_get_microseconds_per_tick());
36369 }
36370 4001659c:   11 00 03 d0     sethi  %hi(0xf4000), %o0
36371 400165a0:   90 12 22 40     or  %o0, 0x240, %o0 ! f4240 <_TLS_Alignment+0xf423      f>
36372 400165a4:   03 10 00 6e     sethi  %hi(0x4001b800), %g1
36373 400165a8:   81 80 20 00     wr  %g0, %y
36374 400165ac:   c4 00 62 84     ld  [ %g1 + 0x284 ], %g2
36375 400165b0:   01 00 00 00     nop
36376 400165b4:   01 00 00 00     nop
36377 400165b8:   84 72 00 02     udiv  %o0, %g2, %g2
36378 400165bc:   81 c3 e0 08     retl
36379 400165c0:   90 10 00 02     mov  %g2, %o0
36380
36381 400165c4 <atexit>:

fsrfsbitmap01.exe  92 bytes

54655 40022d38 <TOD_TICKS_PER_SECOND_method>:
54656
54657 uint32_t TOD_TICKS_PER_SECOND_method(void)
54658 {
54659   return (TOD_MICROSECONDS_PER_SECOND /
54660       rtems_configuration_get_microseconds_per_tick());
54661 }
54662 40022d38:   11 00 03 d0     sethi  %hi(0xf4000), %o0
54663 40022d3c:   90 12 22 40     or  %o0, 0x240, %o0 ! f4240 <_TLS_Alignment+0xf423      f>
54664 40022d40:   03 10 00 c0     sethi  %hi(0x40030000), %g1
54665 40022d44:   81 80 20 00     wr  %g0, %y
54666 40022d48:   c4 00 62 a0     ld  [ %g1 + 0x2a0 ], %g2
54667 40022d4c:   01 00 00 00     nop
54668 40022d50:   01 00 00 00     nop
54669 40022d54:   84 72 00 02     udiv  %o0, %g2, %g2
54670 40022d58:   81 c3 e0 08     retl
54671 40022d5c:   90 10 00 02     mov  %g2, %o0
54672 40022d60:   40 02 30 04     call  400aed70 <__end+0x77d70>
54673 40022d64:   40 02 2f 70     call  400aeb24 <__end+0x77b24>
54674 40022d68:   40 02 2f 64     call  400aeaf8 <__end+0x77af8>
54675 40022d6c:   40 02 2f 58     call  400aeacc <__end+0x77acc>
54676 40022d70:   40 02 2f 4c     call  400aeaa0 <__end+0x77aa0>
54677 40022d74:   40 02 2f 44     call  400aea84 <__end+0x77a84>
54678 40022d78:   40 02 2f 38     call  400aea58 <__end+0x77a58>
54679 40022d7c:   40 02 2f 2c     call  400aea2c <__end+0x77a2c>
54680 40022d80:   40 02 2f 20     call  400aea00 <__end+0x77a00>
54681 40022d84:   40 02 2f 18     call  400ae9e4 <__end+0x779e4>
54682 40022d88:   40 02 2f 0c     call  400ae9b8 <__end+0x779b8>
54683 40022d8c:   40 02 2f 00     call  400ae98c <__end+0x7798c>
54684 40022d90:   40 02 2e f4     call  400ae960 <__end+0x77960>
54685
54686 40022d94 <rtems_rfs_dir__hash>:
{% endhighlight %}

As can be seen above a jump table is added to the symbol
TOD_TICKS_PER_SECOND_method in fsrfsbitmap01.exe which creates the byte
size discrepancy as the code that is processing this is looking for the next
symbol to start to finalise the current symbol as the (new symbols address - 1).

{% highlight c++ %}
448       * Look for the start of a symbol's objdump and extract
449       * offset and symbol (i.e. offset <symbolname>:).
450       */
451       items = sscanf(
452         line.c_str(),
453         "%x <%[^>]>%c",
454         &offset, symbol, &terminator1
455       );
456
457      /*
458       * If all items found, we are at the beginning of a symbol's objdump.
459       */
460       if ( (items == 3) && (terminator1 == ':') ) {
461
462         endAddress = executableInformation->getLoadAddress() + offset - 1;
463
464        /*
465         * If we are currently processing a symbol, finalize it.
466         */
467         if ( processSymbol ) {
468           finalizeSymbol(
469             executableInformation,
470             currentSymbol,
471             startAddress,
472             endAddress,
473             theInstructions
474           );
475         }
476
477        /*
478         * Start processing of a new symbol.
479         */
480         startAddress = 0;
481         currentSymbol = "";
482         processSymbol = false;
483         theInstructions.clear();
484
485        /*
486         * See if the new symbol is one that we care about.
487        \*/
488         if ( SymbolsToAnalyze->isDesired( symbol ) ) {
489           startAddress = executableInformation->getLoadAddress() + offset;
490           currentSymbol = symbol;
491           processSymbol = true;
492           theInstructions.push_back( lineInfo );
493         }
494       }
{% endhighlight %}

The other processing statement detects regular instruction lines from the body
of a symbol.

{% highlight c++ %}
      else if (processSymbol) {

        // See if it is the dump of an instruction.
        items = sscanf(
          inputBuffer,
          "%x%c\t%*[^\t]%c",
          &instructionOffset, &terminator1, &terminator2
        );

        // If it looks like an instruction ...
        if ((items == 3) && (terminator1 == ':') && (terminator2 == '\t')) {

          // update the line's information, save it and ...
          lineInfo.address =
           executableInformation->getLoadAddress() + instructionOffset;
          lineInfo.isInstruction = true;
          lineInfo.isNop         = isNop( inputBuffer, lineInfo.nopSize );
          lineInfo.isBranch      = isBranchLine( inputBuffer );
        }

        // Always save the line.
        theInstructions.push_back( lineInfo );

{% endhighlight %}

The problem was to come up with a check that was restrictive enough to catch all
jump tables but not too retrictive that it cuts symbols short. It looked as if
checking just for '\__end' would be good enough and it did chop the jump tables
at the correct location but I later realised that the branch information had
disappeared in the report. It must have been too loose and chopping lines up in
other places as well.

There were also other jump tables that didn't have this structure so what ended
up being universal was 'call' and '+0x' were all found in all tables. Along with
the normal checks for an instruction line, if all 5 were found it was a jump
table. The symbol could then be finalised as normal. We also needed to set
processSymbol = false, which keeps track if we are in the middle of a symbol
or finished with it. Then along with the 5 items checked, we check processSymbol
to make sure only one line of the jump table is processed and then nothing is
done until a new symbol starts again.

The solution consists of this scanning check for the five items mentioned.

{% highlight c++ %}
436      /*
437       * See if it is a jump table.
438       */
439       found = sscanf(
440         line.c_str(),
441         "%x%c\t%\*[^\t]%c%s %*x %*[^+]%s",
442         &instructionOffset, &terminatorOne, &terminator2, instruction, ID
443       );
444       call = instruction;
445       jumpTableID = ID;
{% endhighlight %}

and then the multiple check if statement which looks complicated but it's as
simple as I could get it.

{% highlight c++ %}
495      /*
496       * If it looks like a jump table...
497       */
498       else if ( (found == 5) && (terminatorOne == ':') && (terminator2 == '\t')
499                && (call.find( "call" ) != std::string::npos)
500                && (jumpTableID.find( "+0x" ) != std::string::npos)
501                && processSymbol )
502       {
503
504           endAddress = executableInformation->getLoadAddress() + offset - 1;
505
506          /*
507           * If we are currently processing a symbol, finalize it.
508           */
509           if ( processSymbol ) {
510             finalizeSymbol(
511               executableInformation,
512               currentSymbol,
513               startAddress,
514               endAddress,
515               theInstructions
516             );
517           }
518           processSymbol = false;
519       }
{% endhighlight %}

In the end the solution was simple enough but there was a lot of sleuthing in
gdb checking variables, following the program around and many, many iterations
of the above solution that did not work.

All error messages are cleared up now and I've gone through a quick code review
with simple changes that I implemented. A more in depth second review of which
I have fixed all issues, one of the major blockers here was the use of symlink
and copying files as the methods are not portable with Linux, BSD and Windows.
I'm working on a third review at the moment, some of which are large changes
that will have to be implemented post-GSOC. I'm hopeful that the critical
changes are doable in the time left and the code will get merged soon.
