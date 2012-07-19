---
layout: post
title: "First Time Building( and Launching) Calligra Suite on OS X"
description: ""
category: "Porting to Mac"
tags: ["C++", "KDE4", "OS X", "Qt 4", "patch", "clang", "cmake"]
---
{% include JB/setup %}

First this post is not a guide to build [Calligra Suite](http://www.calligra.org/) 
on Mac OS X. However, it is for programmers who are interested in porting good
free software to different platforms. Although most part of this suite is still 
under heavy development, one application, [Krita](http://www.krita.org/), is really
in good quality. If you are not satified in [this image manipulation program](http://www.gimp.org/), 
or you would like to paint your creative digital drawings with free software,
you may give Krita a chance.

Unfortunately this suite has not been packaged by the team and [this post](http://forum.kde.org/viewtopic.php?f=137&t=95368)
is talking about the Mac OS port. Thus I'd like to share my porting experience and some **patches** needed to 
build the [Calligra 2.4.3 Release](http://www.calligra.org/news/calligra-2-4-3-released/) on a late-2011 MBP
with latest 10.7.4 OS X patch and using Clang compiler provided by Apple's XCode in (the default) 64bit mode. 
These [patches](#patch) are very likely also adaptable for other platforms.

## Preparation

I use [MacPorts](http://www.macports.org/) for my package management, so for other package 
management system users, searching similiar names and/or reading the description are good 
start point.

Fortunately [Offical guide for building on Unix and similars](http://community.kde.org/Calligra/Building) 
is nearly 100% correct procedure on my platform. Only some notes need to be added.

In the preparation section, just follow what *recommended setup* subsection said and get the source code(This is what I used:
[calligra-2.4.3.tar.bz2](http://download.kde.org/stable/calligra-2.4.3/calligra-2.4.3.tar.bz2), the latest release)

## BUILD/RUNTIME REQUIREMENTS

The second section is about build/runtime requirements. This part is mainly about searching and description reading. 
The required `kdelibs` and `kdebase/runtime`, as in *updated* macports, is:

	kdelibs4 kde4-runtime

Just install these two may give you a minimal develop environment for calligra(other packages will be likely to be also
 installed by the dependency chain). We can check it, for example, if we want to know required `lcms` installed or not,
just type(Of course we should know the correct package name)

	port installed lcms

The procedure of installing these requirement may be very long if starting from a fresh macports install. 
In some case macports ships broken packages, which may be fixed easily or really difficult to solve. The
KDE4 shipped by macports hadn't worked for a long time but **finally** this time it works. After installation 
I looked at the patches and found a workaround for a crash.

For me, since I used Qt 4 for a long time, this procedure was not quite long but took hours. Most unacceptable
part is the fetching and building process of a object relational database server, which is required by the
KDE desktop search feature(Oh, I do not want a second spotlight...)

Then install required packages for some of the applications. I want to build all the applications, so I managed 
it using MacPorts(some of which have been installed, happy installing:) ). After that, the guide instructs us 
to use cmake to config the suite, and see what is not available.

## BUILDING

(I'm using Qt 4.8.2, so bug mentioned in the guide does not hurt me)

Now adjust the cmake command line provided by the offical guide and configure again and again :)

+ pass `-DCMAKE_PREFIX_PATH=/opt/local/` to cmake command line since cmake
prefers things in `/usr/`, however some common libraries' newer version and most *required* 
libraries are installed in `/opt/local` by macports, which may results bad results(eg, warn about
sqlite version, link error due to iconv interface...).
+ set compiler to clang use cmake command line or environment(set to GNU's gcc 
may not result well since macports uses clang to build qt and KDE).
+ use macports to install missing optional packages, finally I installed almost all I can
find in macports, leave (most noticably )OpenGTL and its relatives not installed (I knew this will 
reduce Krita's functionality).
+ macports ships GLEW in Unix fashion, `libGLEW.dylib`(and with pkg-config!), not GLEW.framework. 
The proper fix is to change the file `cmake/modules/FindGLEW.cmake`, but I am lazy. Change some 
lines in the `CMakeCache.txt` and reconfigure it.

I think I have shared most configure-time issues here. Then type `make`, ...mmm...error occurs.

## PATCH

This section is about some errors and patches during the build.
I am very sad some important error are triggered at link time, not compile time.
For immediate use, I posted the patch at [this gist](https://gist.github.com/3138181)
See also [launching](#launching) for 'quick test' results.

1. A typo in `calligra-2.4.3/kexi/kexiutils/utils.h`, this can be fixed
    {% highlight diff %}
diff -r -u calligra-2.4.3.orig/kexi/kexiutils/utils.h calligra-2.4.3/kexi/kexiutils/utils.h
--- calligra-2.4.3.orig/kexi/kexiutils/utils.h  2012-06-26 12:21:31.000000000 +0800
+++ calligra-2.4.3/kexi/kexiutils/utils.h       2012-07-16 16:34:08.000000000 +0800
@@ -537,7 +537,7 @@
     typename QHash<Key, T>::iterator insertMulti(const Key& key, const T& value) {
         return QHash<Key, T>::insertMulti(key.toLower(), value);
     }
-    const Key key(const T& value, const Key& defaultKey) const {
+    const Key key(const T& value, const Key& key) const {
         return QHash<Key, T>::key(value, key.toLower());
     }
     int remove(const Key& key) {
{% endhighlight %}

2. In `calligra-2.4.3/plan/plugins/schedulers/rcps/libs/`, many files include `malloc.h`, which is not
available on OS X. I just changed these to `stdlib.h`, everything works fine. Is there anything other 
than `malloc` called?

3. link time error about module *wpgimport*. After investigating `cmake/modules/FindWPG.cmake`, I decided to
    {% highlight diff %}
diff -r -u calligra-2.4.3.orig/filters/karbon/wpg/CMakeLists.txt calligra-2.4.3/filters/karbon/wpg/CMakeLists.txt
--- calligra-2.4.3.orig/filters/karbon/wpg/CMakeLists.txt       2012-06-26 12:18:00.000000000 +0800
+++ calligra-2.4.3/filters/karbon/wpg/CMakeLists.txt    2012-07-16 18:51:31.000000000 +0800
@@ -5,7 +5,7 @@
    
 kde4_add_plugin(wpgimport ${wpgimport_PART_SRCS})
    
-target_link_libraries(wpgimport komain ${LIBWPG_LIBRARIES} ${LIBWPG_STREAM_LIBRARIES} ${WPD_LIBRARIES})
+target_link_libraries(wpgimport komain ${LIBWPG_LIBRARIES} ${LIBWPG_STREAM_LIBRARY} ${WPD_LIBRARIES})
    
 install(TARGETS wpgimport DESTINATION ${PLUGIN_INSTALL_DIR})
 install(FILES karbon_wpg_import.desktop DESTINATION ${SERVICES_INSTALL_DIR})
{% endhighlight %}
    It fixed the problem but I'm not quite sure about this, since in `cmake/modules/FindWPG.cmake`

        SET(LIBWPG_LIBRARIES ${LIBWPG_LIBRARY} ${LIBWPG_STREAM_LIBRARY})

    This line not working as it is?

4. Link time error about module *excelimport*, *excelexport* and *excelimporttodoc* filters. This problem
is somewhat hard to find. The linker says undefined reference to QSharedPointer::deref, but where does the code
call it? May be the assignment operator -- In `filters/sheets/excel/sidewinder/objects.cpp`
    {% highlight c++ %}
void OfficeArtObject::setText(const TxORecord &text)
{
    m_text = text;
}
{% endhighlight %}
    Since when compiling lots of warning flushed the screen I removed compiled object files related to this module and 
made clang++ to compile them again. Oh! yes, the clang compiler correctly warned: `call delete to an incomplete
type QTextDocument`. I still had to find why link-time error occurred. Finally I realized this behavior was
related to the treatment to `(static) inline` of the clang compiler. Look at `QSharedPointer`, `deref` method 
and related are inlined. 

    Thanks to the clang compiler, I got this patch.
    {% highlight diff %}
diff -r -u calligra-2.4.3.orig/filters/sheets/excel/sidewinder/excel.h calligra-2.4.3/filters/sheets/excel/sidewinder/excel.h
--- calligra-2.4.3.orig/filters/sheets/excel/sidewinder/excel.h 2012-06-26 12:18:00.000000000 +0800
+++ calligra-2.4.3/filters/sheets/excel/sidewinder/excel.h      2012-07-16 19:24:19.000000000 +0800
@@ -38,7 +38,7 @@
 #include "ODrawToOdf.h"
 #include "pictures.h"

-class QTextDocument;
+#include <QTextDocument>

 namespace Swinder
 {
{% endhighlight %}

## LAUNCHING

Before launching, dbus should be loaded, macports issues a note about launching dbus after your every KDE4 
package installation. For example,

    launch load -wF /Library/LaunchAgents/org.freedesktop.dbus-session.plist

This (permanently) enables starting dbus after login.

Now pay attention to the directory structure, applications are installed in 
`/Applications/KDE4/` as Mac OS X native app, and also `$KDEHOME` changes from `$HOME/.kde` to `$HOME/Library/Preferences/KDE/`.
Besides these, the official guide's *Running Calligra applications* section works. For most Mac users,
being able to click the app to run the applications is really favorable.

Now I can run Krita. Yes, Krita works fine. After several minutes testing about the whole suite 
(only first glance really):

+ A CRASH SHARED BY ALL APPs. REPRODUCE is easy: open any calligra app, new a document, edit something, then close 
the document or quit the application. The application will ask whether save, discard or cannel. Choose discard. 
Then the application crashes with message: 

      QKqueueFileSystemWatcherEngine: error during kevent wait: Bad file descriptor

   Following report items **ignore** this crash.
+ Krita works
+ Karbon works
+ words works (sorry I really want to say this app is far from feature complete)
+ stage works
+ sheets works
+ Kexi's start menu display corrupts(seems the new UI is not finished?)
+ plan not work, crashes when starting
+ flow works, stencil box has some minor display problem
+ Assistant tools work, kthesaurus.app, visualimagecompare.app, calligraconverter.app, etc seem OK at command line
+ [This ticket](https://git.reviewboard.kde.org/r/102147/), for all apps of KDE4, waiting for it...
+ On OS X, default shared memory limit is too slow, only 4MiB. Editing `/etc/sysctl.conf` may be necessary for 
some features work.

Also Krita says it cannot get my physical memory size and use 1GiB, though I can fix this issue but I just don't 
want to do this since currently I don't have much memory for this app...(hint: sysctl hw.memsize)

## ACKNOWLEDGEMENT

Sorry for my long article and poor markdown skill. Feel free to comment. 
Names or trademarks are owned by coresponding author(s)/orgnazition(s). Use of these names with best regards and wishes.
