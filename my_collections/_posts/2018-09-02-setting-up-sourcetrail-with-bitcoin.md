---
layout: post
title:  "Setting Up Sourcetrail with Bitcoin"
---

I was listening to a [recent CppCast podcast episode](http://cppcast.com/2018/08/eberhard-grather/) where the guest, 
Eberhard Gräther, described the software he built: "Sourcetrail, a cross-platform source explorer for faster 
understanding of unfamiliar source code".

The visualization feature of Sourcetrail caught my attention as I enjoy using a similar 
[call graph feature in Doxygen](https://www.stack.nl/~dimitri/doxygen/manual/diagrams.html).

I [downloaded Sourcetrail](https://www.sourcetrail.com/downloads). Sourcetrail relies on a `compile_commands.json` file 
to automatically identify all the source files that need to be indexed. This file can be exported with CMake but Bitcoin uses 
Autotools for its build system. After attempting a number of different solutions<sup>[1](#myfootnote1)</sup> the 
resulting `compile_commands.json` only included 9 source files. If you're able to do better with an automated tool, let 
me know! Instead I manually set up Sourcetrail:

Sourcetrail setup:
1. Open Sourcetrail
2. Click the `New Project` button
3. `Sourcetrail Project Name` can be anything, I used bitcoin
4. `Sourcetrail Project Location` can be anywhere, I used ~/src/sourcetrail/
5. Click the `Add Source Group` button
6. Click the `Empty C++ Source Group` and `Next` buttons
7. Change the `C++ Standard` to `c++11` and click `Next`
8. In `Files & Directories to Index` add your `bitcoin/src` directory and click `Next`
9. Exclude `bitcoin/src/leveldb`
10. Click `auto-detect` and `Start` for `Include Paths`, click the pencil icon and paste in the following
(if you don't know what version to use, search in `bitcoin/Makefile` after running `./autogen.sh && ./configure`):
    ```
    /usr/local/Cellar/openssl/1.0.2p/include
    /usr/local/Cellar/libevent/2.1.8/include
    /usr/local/opt/berkeley-db@4/include
    /usr/local/Cellar/qt/5.11.1/include
    /usr/local/Cellar/qt/5.11.1/include/QtNetwork
    /usr/local/Cellar/qt/5.11.1/include/QtWidgets
    /usr/local/Cellar/qt/5.11.1/include/QtGui
    /usr/local/Cellar/qt/5.11.1/include/QtCore
    /usr/local/Cellar/qt/5.11.1/include/QtDBus
    /usr/local/Cellar/qt/5.11.1/include/QtTest
    ```
11. Click `detect` for `Global Include Paths` and the `Next` button
12. For `Global Framework Search Paths` click the `detect` button and `Save`
13. No compiler flags, click `Next`
14. Click `Create`
15. Click `Start` for the indexing

All done!




_footnote_
<a name="myfootnote1">1</a>. Build and install [Bear](https://github.com/rizsotto/Bear) or 
[scan-build](https://github.com/rizsotto/scan-build). On macOS they do not work with the default `make` due to a 
system security policy. To work around this, `brew install make && pip install scan-build` 
and run `PATH="/usr/local/opt/make/libexec/gnubin:$PATH" && ./autogen.sh && ./configure  && gmake clean && intercept-build --override-compiler --append gmake -j8` in your bitcoin directory.

