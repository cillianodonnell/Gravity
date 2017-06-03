---
layout: post
title:  "Design Stories : Gravity"
date:   2017-05-05 19:45:31 +0530
categories: ["rtems", "dev"]
author: "Cillian O'Donnell"
---
I am a 2nd Year Electronic & Computer Engineering student at Dublin City University and I will be improving the RTEMS coverage analysis tools for GSOC 2017. This blog will document the development process, all major milestones and design decisions will be recorded here.

This project will switch the RTEMS Source Builder (which is a program that ensures a consistent build process for all users, instead of each using their own build scripts) from QEMU simulator to a QEMU variant specially modified for coverage analysis, Couverture-QEMU which includes more detailed execution trace data for analysis. I will integrate Couverture-QEMU and the scripts driving it into RTEMS Tester framework, converting the shell scripts to Python. The coverage report tool Covoar written in C++ will then be modified to generate XML output, it is currently generating HTML. The format for the XML report will be based on feedback solicited from the RTEMS community on the devel mailing list.
