# Developing LibreOffice Extensions in C++.
This is a ongoing series of posts on developing C++ LO extensions mainly for Calc. It is heavily based on https://wiki.openoffice.org/wiki/Documentation/DevGuide/OpenOffice.org_Developers_Guide but with more emphasis on starting early with C++ and Calc through complete code examples. The complete code for every examples mentioned in the blog are meant for GNU/Linux systems. It might also work on MS Windows and MacOSX but we do not support those yet.


## Types of audience and their prerequisites

1. Developers who wants to understand how to write LO extensions in C++. We refer to this type of audience as **AUD-A**.

   Prerequisites : A fair grasp of C++, Linux based build systems using cmdline. The reader must first build LO from source code. If you don't know how to do this yet, first please follow the article at [devsetup](../misc/devsetup.md). Then must follow instructions at [SDKSetup](../misc/sdksetup.md) to setup LO SDK. The reader needs to start from [Part1](part1.md). Unless otherwise indicated they are the default audience for all content.

2. Developers who wants to build our extensions from source without understanding the code. We refer to this type of audience as **AUD-B**.

   Prerequisites : A fair grasp of Linux based build systems using cmdline. The reader must first build LO from source code. If you don't know how to do this yet, first please follow the article at [devsetup](../misc/devsetup.md). Then must follow instructions at [SDKSetup](../misc/sdksetup.md) to setup LO SDK. The reader can start from any part of the blog series.
   
3. Users/Testers who wants to just want to run/test our extensions. We refer to this type of audience as **AUD-C**.

   Prerequisites : Access to a **Fedora 24 Linux 64 bit** box (as we supply prebuilt extensions only for this distro), LibreOffice Calc version >= 5.1. You can skip Part1, Part2 and Part3 as they do not create extensions. [Part4](part4.md) is your starting point.



Regardless of the type of audience, the reader may want to read the [Pre-FAQ](prefaq.md) to get some general understanding of the way LO-extensions behave.