---                                                                             
layout: post                                                                    
title:  "Testing Couverture-QEMU"                                    
date:   2017-06-12 13:51:31                                              
categories: rtems, testing                                                     
--- 
The RTEMS Tester work using coverage has been left aside for the moment. This
post details the testing of Couverture-QEMU against the RSB built QEMU without
the coverage flag being passed to rtems-test.

The Xilinx Zynq A9 BSP is built with:

{% highlight bash %}
$HOME/development/rtems/kernel/rtems/configure \
--prefix=$HOME/development/rtems/4.12 \
--target=arm-rtems4.12 \
--enable-rtemsbsp=xilinx_zynq_a9_qemu \
--enable-tests
make
make install
{% endhighlight %}

When running the testsuite with rtems-test it produces qemu.cfg errors as
before. These error messages produce exit codes that aren't standardized and
are more of a general error message which is not much to go on. I found a useful
flag --report-mode=all to add to rtems-test runs that produces a more detailed
error log. This produced:

{% highlight bash %}
(process:30369): GLib-WARNING ** /build/glib2.0-prJhLS/glib2.0-2.48.2/./glib/    gmem.c:483: custom memory allocation vtable not supported
qemu: hardware error: gic_cpu_write: Bad offset 100 

CPU #0: 
R00=400001d3 R01=00000000 R02=00000000 R03=f8f00200  
R04=400001d3 R05=00000000 R06=00000000 R07=00100fe0
R08=00000000 R09=00000000 R10=00000000 R11=00000000 
R12=00000000 R13=00100fe0 R14=00000705 R15=000006c8   
PSR=600001f3 -ZC- T svc32
{% endhighlight %}

Now there was a neccessary patch to get Couverture-qemu to build from the
previous work that swapped 'gthread-2.0' for 'glib-2.0' that might be a problem.
I also found that the problem first arose in glib-2.46.0 so maybe forcing the 
use of an earlier version would work. I found some patches from mainline qemu
to fix the GLib warning. It was caused because GLib removed support for malloc
vtable because it broke when constructors called g_malloc. The patches fix was
to remove malloc tracing, I was worried that this might have negative effects
on the coverage analysis ability as it reduces trace information. After this I 
am still left with the 'gic_cpu_write..' problem.

Luckily I got talking to one of the Couverture-Qemu devs and he told me that
they hadn't updated the default branch for the source download. So there was
actually a newer stable release which fixed all of the above problems. If I 
wasn't such a git novice I probably would have figued this out myself. I had 
been using 'git remote show' and thought this is showing me what was available
but I needed 'git ls-remote' to see all the possible remote branches. There was
'ref/heads/qemu-stable-2.4.1' and in order to retrieve that I needed 
'git checkout -b qemu-stable-2.4.1 origin/qemu-stable-2.4.1'. All of the above
problems disappeared and none of the patches were now needed. It did introduce
a few new dependencies for my system 'libpixman-1-dev' and 'libfdt-dev'

The testing consisted of running hello.exe by hand and passing different options
to it to determine the correct options and then running the testsuite with
RTEMS Tester for both RSB-QEMU and Couverture-Qemu. Adjust the qemu options if
most results were invalid or timed out. If a BSP had no current support in
RTEMS Tester I would add a bsp.mc file to handle the qemu options.

There were 3 BSPS that produced good results almost entirely agreeing between
RSB-Qemu and Couverture-Qemu.

1. ARM: xilinx zynq a9 qemu
2. ARM: realview pbx a9 qemu
3. SPARC: Leon 3

{% highlight bash %}

/**************************
    xilinx_zynq_a9_qemu
/**************************


qemu-system-arm -no-reboot -monitor null -serial stdio -nographic -M xilinx-zynq-a9 -kernel...

/----------------------------

  COUVERTURE-QEMU

/----------------------------

/---------------
  SAMPLES
/---------------


Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:00.559658
Testing time     : 0:00:07.275561

/-------------
    FULL
/-------------

Passed:        551
Failed:          5
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         2
Invalid:         1
Total:         566
Average test time: 0:00:00.652736
Testing time     : 0:06:09.448995


/------------------

    RSB-QEMU

/------------------ 

qemu-system-arm -no-reboot -serial null -serial mon:stdio -nographic -M xilinx-zynq-a9 -kernel...

/---------------
    SAMPLES
/---------------

Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:00.560756
Testing time     : 0:00:07.289829

/-------------
    FULL
/-------------

Passed:        545
Failed:          4
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         9
Invalid:         1
Total:         566
Average test time: 0:00:01.168632
Testing time     : 0:11:01.445923



   realview_pbx_a9_qemu

/-----------------------

    RSB-QEMU

/--------------------------

qemu-system-arm -no-reboot -monitor null -serial stdio -nographic -M realview-pbx-a9 -kernel...

/---------------
    SAMPLES
/---------------

Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:01.656712
Testing time     : 0:00:21.537256

/--------------
    FULL
/--------------

Passed:        547
Failed:          3
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         8
Invalid:         1
Total:         566
Average test time: 0:00:01.175699
Testing time     : 0:11:05.445813

/------------------------

    COUVERTURE-QEMU

/------------------------

qemu-system-arm -no-reboot -monitor null -serial stdio -nographic -M realview-pbx-a9 -kernel...

/---------------
    SAMPLES
/---------------


Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:00.617387
Testing time     : 0:00:08.026043

/--------------
    FULL
/--------------

Passed:        553
Failed:          4
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         1
Invalid:         1
Total:         566
Average test time: 0:00:00.671176
Testing time     : 0:06:19.886106

/************
   Leon 3 
/************

qemu-system-sparc -no-reboot -monitor null -serial stdio -nographic -net none -M leon3_generic -kernel...

/--------------------------

    COUVERTURE-QEMU

/--------------------------

/---------------
    SAMPLES
/---------------


Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:03.679456
Testing time     : 0:00:47.832933

/-------------
    FULL
/-------------

Passed:        543
Failed:          3
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         9
Invalid:         3
Total:         565
Average test time: 0:00:01.693401
Testing time     : 0:15:56.771872

/---------------------------

    RSB-QEMU

/---------------------------

/---------------
    SAMPLES
/---------------


Passed:        11
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        0
Invalid:        1
Total:         13
Average test time: 0:00:03.679446
Testing time     : 0:00:47.832800

/-------------
    FULL
/-------------

Passed:        544
Failed:          2
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:         9
Invalid:         3
Total:         565
Average test time: 0:00:01.739428
Testing time     : 0:16:22.777222
{% endhighlight %}

For the folowing 4 BSPS qemu runs the tests but fails to exit and so the tests
time out. If I run hello.exe by hand it hangs on the line 'end of test hello
world' and must be manually exited (crtl+c). This turns into tests timing out
in RTEMS Tester.

{% highlight bash %}
* BEGIN OF TEST HELLO WORLD *
Hello World
* END OF TEST HELLO WORLD *
qemu: terminating on signal 2
{% endhighlight %}

The BSPS affected are

ARM: lm3s6965_qemu

i386: PC386 

LM32: lm32_evr

MIPS: malta

A note on PC386, Couverture-Qemu reacts as above but passes the test instead of
a timeout. RSB-Qemu hangs just before the hello world output and must be 
manually exited. When hello.exe is run both take 3 minutes, RSB-Qemu times out
and Couverture-Qemu passes it. I ran the entire testsuite with Couverture-qemu
because it appeared to have valid results i.e not many invalid or timed out.
It took 7 hours and a lot of tests passed that probably should have timed out.

{% highlight bash %}
/-------------------------------------

    COUVERTURE-QEMU

/-------------------------------------

qemu-system-i386 -m 128 -boot b -hda path_to/rtems-boot.img -no-reboot -monitor null -serial stdio -nographic -append "--console=com1;boot;"

/--------------
    SAMPLES
/--------------

Passed:         9
Failed:         0
User Input:     1
Expected Fail:  0
Indeterminate:  0
Benchmark:      0
Timeout:        2
Invalid:        1
Total:         13
Average test time: 0:00:14.133522
Testing time     : 0:03:03.735787

/------------
    FULL
/------------

Passed:        485
Failed:          0
User Input:      4
Expected Fail:   0
Indeterminate:   0
Benchmark:       3
Timeout:        74
Invalid:         0
Total:         566
Average test time: 0:00:44.922802
Testing time     : 7:03:46.305933
{% endhighlight %}


There was 2 other exceptions lm32: milkymist BSP fails to run hello.exe due to
an audio driver issue

{% highlight bash %}
qemu-system-lm32 -no-reboot -monitor null -serial stdio -nographic -net none -M milkymist -kernel $HOME/development/rtems/milkymist/lm32-rtems4.12/c/milkymist/testsuites/samples/hello/hello.exe
audio: Could not init `oss' audio driver
Warning: nic milkymist-minimac2.0 has no peer
{% endhighlight %}

The other was PowerPC: qemuprep which qemu fails to boot as no boot partition is
found. I tried to boot it from a floppy disk image with

{% highlight bash %}
qemu-system-ppc -boot a -fda ./rtems-boot.img -m 20MB -append "console=/dev/com1" -serial stdio -no-reboot -kernel $HOME/development/rtems/qemuprep/powerpc-rtems4.12/c/qemuprep/testsuites/samples/hello/hello.exe
{% endhighlight %}

It still found no boot partition. I filed tickets for all these issues [1],
[2] and [3]. I now have 3 BSPS to build my RTEMS Tester coverage work on and to
test with. The other BSPS can be sorted out later on. I will now begin the work
of switching the RTEMS Source Builder to build Couverture-Qemu as the standard
RTEMS QEMU variant.

[1]: https://devel.rtems.org/ticket/3039
[2]: https://devel.rtems.org/ticket/3038
[3]: https://devel.rtems.org/ticket/3037
