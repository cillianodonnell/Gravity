---
layout: post
title:  "GSOC: Final Report"
date:   2017-08-27 10:04:00
categories: rtems, gsoc
---
## Introduction ##
I am the 3rd student to work on this coverage tools project, a lot of work had
already been done previously. The work was never merged and as such had fallen
into disrepair. My work revived the coverage tools project to generate reports
again and made improvements to covoar so that it is in better working order and
closer to merging that it had been before. I also developed a build for
Couverture-QEMU so the RTEMS Source Builder(RSB) can automatically setup and
build this version of qemu in a way that is tested and known to work with RTEMS.

## RTEMS Source Builder ##
The RSB build consists of 3 files, the blueprint of which was already present
in the current qemu build.

* A general config file in source-builder/config. This contains the general
configure and build instructions (for say version 2.x.x release).

[couverture-qemu-2-1.cfg](https://github.com/cillianodonnell/rtems-source-builder/blob/qemu_switch/source-builder/config/couverture-qemu-2-1.cfg)


* More specific version config file in bare/config/devel which contains the
source location and any patches that need to be applied before configuring
(for say version 2.4.1)

[couverture-qemu-git-1.cfg](https://github.com/cillianodonnell/rtems-source-builder/blob/qemu_switch/bare/config/devel/couverture-qemu-git-1.cfg)


* A .bset in bare/config/devel which specifies all the dependencies and the
order in which to build them.

[couverture-qemu.bset](https://github.com/cillianodonnell/rtems-source-builder/blob/qemu_switch/bare/config/devel/couverture-qemu.bset)


This build is working, tested and ready for use since the end of June.
However there is a set of 3 patches for Leon3 from Gaisler that would be
applied in couverture-qemu-git-1.cfg file above. I have been working to get
those patches merged into upstream QEMU master with an Ada Core developer
Frederic Konrad. I'm waiting to submit this RSB build to RTEMS devel list until
the Gaisler patches are merged upstream. I believe the ideas from the patches
were already applied to Couverture-QEMU but for some reason never made it into
upstream QEMU but just to be sure I'll wait until its merged and test the 2
qemu's against each other.

## Coverage Analysis Tools ##
The coverage tools are working very well, they generate reports for the
collective coverage of the testsuite run against a set of symbol libraries
chosen by the user. The tools can be built and used from my github here.

[Build covoar and run RTEMS Tester from here](https://github.com/cillianodonnell/Final-GSOC)
(For details on how to build and use the tools, see documentation section below)

Much of my time was spent just getting the coverage tools to run again like they
used to in 2014 and 2015. Much of the difficulty here lay in my inexperience.
Figuring out how RTEMS Tester works, how covoar works, learning git,
learning python. In general just managing the complexity of all the tools
working together, how they are interacting combined with very little
documentation from the previous students.

After I had things working again, there are 3 main fixes that account for my
main contributions. The 3 fixes are detailed below.

* There was a check for if the qemu trace branch equaled the size of the symbol
from the coverage map. It was deemed to be too restrictive a check and removed.
This is a very simple fix but there was a lot of detective work in GDB to make
that decision. I also had to learn GDB to do it :)

[commit](https://github.com/cillianodonnell/Final-GSOC/commit/4a2975825404aaf391fb36d35640a8acd7bdb490)

* The next problem was some executables had jump tables added to the end of
symbols in their objdump, while others did not add these for the same symbols.
This created a discrepancy in their size when comparing and checking those
symbols. Each executables objdump is processed and symbols are picked out by
regex matching. The process hadn't accounted for these jump tables being
randomly added. My fix for this was finding something suitably specific,
5 things altogether, 3 to determine it is a normal instruction line and not a
new symbol and a further 2 to distinguish a jump table from a regular
instruction.

  {% highlight c++ %}
  436      /*
  437       * See if it is a jump table.
  438       */
  439       found = sscanf(
  440         line.c\_str(),
  441         "%x%c\t%\*[^\t]%c%s %*x %*[^+]%s",
  442         &instructionOffset, &terminatorOne, &terminator2, instruction, ID
  443       );
  444       call = instruction;
  445       jumpTableID = ID;
  {% endhighlight %}

This gathered the line data and then a check for 'call' and '+0x' turned out to
be the common denominator in the different kind of jump tables. The symbol is
finalised and the processSymbol variable set to false so no other lines of the
jump table will be processed.

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

The full commit for this fix is:
[commit](https://github.com/cillianodonnell/Final-GSOC/commit/90f879cc99a3a5141842e1f7b6bf1d0c923ef84f)


* The final fix is that the objdump files used to gather symbol information
would be left lying around in the event of a crash. This was a merge blocker,
the solution is to use rld::process::tempfile for the objdump files and
rld::process::execute to run objdump for the chosen architecture which also
removes the need for a pipe to sed. These classes are part of rtemstoolkit
which is a collection of best practices for a number of procedures written in
C++. I learned to use these and implemented both. Tempfile class is then used
to open, read and write to the files. It has an integrated tempfiles clean up
procedure, which was added with a fatal signals check in covoar, which would
always call the tempfile clean up routine in the event of a crash.

The following is the rewrite of the getFile function in ObjdumpProcessor.cc
which handled the production of the objdump tempfiles. This is the centre of
this patch.

{% highlight c++ %}
-  FILE* ObjdumpProcessor::getFile( std::string fileName )
+  void ObjdumpProcessor::getFile(
+    std::string fileName,
+    rld::process::tempfile& objdumpFile,
+    rld::process::tempfile& err
+    )
   {
-    char               dumpFile[128];
-    FILE*              objdumpFile;
-    char               buffer[ 512 ];
-    int                status;
-
-    sprintf( dumpFile, "%s.dmp", fileName.c_str() );
-
-    // Generate the objdump.
-    if (FileIsNewer( fileName.c_str(), dumpFile )) {
-      sprintf(
-        buffer,
-        "%s -Cda --section=.text --source %s | sed -e \'s/ \*$//\' >%s",
-        TargetInfo->getObjdump(),
-        fileName.c_str(),
-        dumpFile
-      );
-
-      status = system( buffer );
-      if (status) {
-        fprintf(
-          stderr,
-          "ERROR: ObjdumpProcessor::getFile - command (%s) failed with %d\n",
-          buffer,
-          status
-        );
-        exit( -1 );
+    rld::process::status        status;
+    rld::process::arg_container args = { TargetInfo->getObjdump(),
+                                         "-Cda", "--section=.text", "--source",
+                                         fileName };
+    try
+    {
+      status = rld::process::execute( TargetInfo->getObjdump(),
+               args, objdumpFile.name(), err.name() );
+      if ( (status.type != rld::process::status::normal)
+           || (status.code != 0) ) {
+        throw rld::error( "Objdump error", "generating objdump" );
+      }
+    } catch( rld::error& err )
+      {
+        std::cout << "Error while running" << TargetInfo->getObjdump()
+                  << "for" << fileName << std::endl;
+        std::cout << err.what << " in " << err.where << std::endl;
+        return;
       }
-    }
-
-    // Open the objdump file.
-    objdumpFile = fopen( dumpFile, "r" );
-    if (!objdumpFile) {
-      fprintf(
-        stderr,
-        "ERROR: ObjdumpProcessor::getFile - unable to open %s\n",
-        dumpFile
-      );
-      exit(-1);
-    }

-    return objdumpFile;
+    objdumpFile.open( true );
   }
{% endhighlight %}

The full commit for this, which spans over many files:
[commit](https://github.com/cillianodonnell/Final-GSOC/commit/c6be049abac288182d226b7c002ff930ccded0be)

## Mergable ##
Everything that could be committed by myself and the 2 other students I
collected and submitted to the RTEMS devel list [here](https://lists.rtems.org/pipermail/devel/2017-August/018897.html)
This consisted mainly of standalone fixes to covoar, including the above
3 fixes.

The RTEMS Tester integration is working and very close to merging but there are
still a few things that need to change before it is accepted. The main blockers
are as follows.

* nm is used to generate a list of symbols in a file that is referenced as the
symbols of interest for that set. nm is not portable and so must be removed.
The fix for this is use rtemstoolkit and generate the list of symbols from the
ELF files.

[nm use](https://github.com/cillianodonnell/Final-GSOC/blob/coverage-patches/tester/covoar/SymbolSet.cpp#L98)

* There is also a use of addr2line piped to dos2unix to find the source
lines which match the objdump instruction lines. Again we want to limit the use
of external tools that are not portable across platforms.

[addr2line use](https://github.com/cillianodonnell/Final-GSOC/blob/coverage-patches/tester/covoar/DesiredSymbols.cc#L466)

* The trace files that are generated by QEMU as each test is run in RTEMS
Tester need to stay for later use by covoar which runs after the testsuite is
finished. They are currently cleared up by a try finally statement in python,
which works for Ctrl+C exit but probably not SIGTERM. This cannot be handled
the same way as the other tempfiles as they are not generated by covoar. These
files have a .cov ending and are currently generated beside the executable they
match in the build tree.

[current cleanup](https://github.com/cillianodonnell/Final-GSOC/blob/coverage-patches/tester/rt/coverage.py#L380)

* covoar needs to detect target architecture internally (e.g sparc-rtems4.12),
this can be done with get_exec_prefix() from rld-cc.h in rtemstoolkit.


Some further improvements that would be nice to make but not merge blockers are:

* Convert the symbol_sets.cfg file to the INI format. The symbol_sets.cfg file
is currently in a non standard format, created just for it. This file contains
the sets of libraries of symbols and groups them together to be treated as a set
to see what percentage of them the testsuite covers.

* Generate XML reports. Currently the tools generate html and text reports.

* Generate Gcov reports. There is Gcov support in covoar that is close to
working but the gcov output generated is not trusted. The fix for this would
involve vaildating the Gcov output to make sure it is in a format that the Gcov
program accepts.

* Combine how-to documentation below with existing covoar docs to make a ReST
format doc thats integrated with the main RTEMS Tester docs.

## Documentation ##
* The details of how to use the RTEMS Source Builder to build Couverture-QEMU:

[How to Build Couverture-QEMU](https://devel.rtems.org/wiki/GSoC/2017/coveragetools#BuildingCouverture-QemuwiththeRSB)


* Instructions on how to use the coverage analysis tools and generate reports
for symbol sets of your choice:

[How to use the Coverage Tools](https://devel.rtems.org/wiki/GSoC/2017/coveragetools#CoverageAnalysisinRTEMSTester)

Short status updates for each week of the GSOC project:

[Weekly status updates](https://devel.rtems.org/wiki/GSoC/2017#CillianODonnell)

Longer blog posts detailing specific problems and their solutions:

[Development Blog](https://cillianodonnell.github.io/index.html)
