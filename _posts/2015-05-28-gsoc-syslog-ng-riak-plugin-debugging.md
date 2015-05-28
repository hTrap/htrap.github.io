---
layout: post
title: Compiling and Debugging the new Riak Plugin
tags: gsoc syslog-ng coding-period
category: work self
comments: true
authon: Parth Oberoi
meta: Using Gdb
---

To compile the riak plugin that i had written so far along
with syslog-ng source. Initial steps taken were -

+ fork [Syslog-ng](https://github.com/balabit/syslog-ng) repo from GitHub
+ clone it locally
+ create a new branch `f/riak` where f stands for *feature*
+ add the plugin or the code under `syslog-ng/modules/riak/`

A quick `./configure --enable-riak` showed me that the plugin was inactive. To
fix this the `configure.ac` file had to be tweaked, this
was thankfully fixed by my mentor *Gergely Nagy* . Continuing with the
integration , the `make` command threw a few hundred errors I had made in writing the
code which were fixed slowly - taking one at a time.

Since we are creating a
seperate module/library for talking to riak, hence I could not use glib
functions for the part of the code that would be used along with that library,
for eg. `g_new0()` became `calloc`,`g_char` became `char`, etc. But I used
`gboolean` for the parts that
will be used by Syslog-ng and not the library.

After fixing all the bugs I was suggested by my mentor to compile it using
debugging flags and also change the path of installation that is not default,

+ `./configure CFLAGS="-ggdb3 -O0" --prefix=$HOME/install/syslog-ng`
which changed my installation path to *$HOME/install/syslog-ng*
also the use of `sudo ldconfig -V` would index all shared libraries.

For running Syslog-ng I had to just export the path by using ` export
PATH="${HOME}/install/syslog-ng/sbin:$PATH"`


Everything after this went smoothly until I used `syslog-ng -s` the dreaded **segmentation fault**
stared me in the face. Now was the time for gdb. There are a couple of ways you
can use gdb with Syslog-ng.
First is `gdb --args ~/install/syslog-ng/sbin/syslog-ng -sF` which you can use
to `run` under gdb, set breakpoints and start again. The second one is using
the core dumped by the segmentation fault which is also useful in sharing with
the community. By using `syslog-ng -F --enable-core`, generates a file named
core in the working directory and used as `gdb syslog-ng core` and then `bt
full` will give you the full backtrace.

After fixing a few more errors in the `riak-grammar.ym`, the configuration and
the grammar was successfully compiled with Syslog-ng. Now is the time for some
worker-threads.

