---
layout: post
title: "switching C++ on OS X"
description: ""
category: "Using Mac"
tags: ["C++","OS X","gcc","clang"]
---
{% include JB/setup %}

It is well known that OS X < 10.9 uses obsolute gcc 4.2 based libstdc++
and not compatible with newer FSF libstdc++ (e.g. &gt; 4.5) or libc++
by LLVM team.

It looks possible to use clang-3.3 with FSF's libstdc++-4.8
and gcc-4.8 with libc++. I tested it with [MacPorts](http://www.macports.org/)
currently. By using something like

	g++-mp-4.8 -std=c++11 -nostdinc++ -isystem /opt/local/libexec/llvm-3.3/lib/c++/v1/ -lc++
	clang++-mp-3.3 -nostdinc++ -cxx-isystem /opt/local/include/gcc48/c++/ \
          -cxx-isystem /opt/local/include/gcc48/c++/x86_64-apple-darwin11/ -L/opt/local/lib/gcc48

std=c++11 is needed by g++ to parse libc++ while clang works in both 98 and 11 mode with FSF's.
The standard C++ including path used by gcc48 is

	 /opt/local/include/gcc48/c++/
	 /opt/local/include/gcc48/c++/x86_64-apple-darwin11
	 /opt/local/include/gcc48/c++/backward

In my previous command line, I omitted the `backward` directory for simplicity.

In general, mixing libc++ and libstdc++ will simply not link, but mixing
old libstdc++ with new libstdc++ will cause crash.

It is unknown whether gcc compiled code with libc++ will crash when mixing
clang compiled code with libc++.

Thanks
