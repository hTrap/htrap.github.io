---
layout: post
title: Riak client library
tags: gsoc syslog-ng riak protocol-buffers
category: work self gsoc
comments: true
author: Parth Oberoi
meta: Riak PBC API
---

##The Syslog-ng Riak module
The Riak [module](https://github.com/hTrap/syslog-ng/tree/f/riak/modules/riak)
has three worker thread methods -

+ `riak_worker_thread_init` : This method does the sanity checks, i.e. checks 
    if the bucket, key or value is NULL and fails if required values are
    unchecked.
+ `riak_worker_thread_deinit` : This method deferences all the LogTemplates
    in the RiakDestDriver structs and also frees th emomory alloted to the
    `char *` variables of the structure.
+ `riak_worker_insert` : This method is spposed to do the bulk of the work 
    i.e. store messages in riak, but right now we have a printf statement in it
    which is used for making sure that is module enters the insert method.

Then for debuging and testing our module we change the source of the
syslog-ng.conf to `tcp(ip("127.0.0.1") port(12345) flags(no-parse));` .This
listens to port 12345 on localhost and is tested by using `echo foo | nc
127.0.0.1 12345` on the terminal. Before this a `syslog-ng -Fvde` on a
different terminal shows all the messages in th foreground and the foo string
that was echoed shows itself here. Also the message used a printf in the
worker_insert method is displayed here aswell which makes it certain that the
insert method is being used when a message is being recieved by Syslog-ng. This
leaves us with a syslog-ng module that can parse new config options, set up its internal state, 
and can even receive and ignore messages!! \o/

## Riak Client Library

As mentioned in my proposal, the C libraries of riak client that are present on
github are all obsolete and hence I have to write a library that can talk to Riak, 
and supports enough of the PBC protocol, that our two use cases(set and store) are covered.
The library so far recides [here](https://github.com/algernon/riack). It is
licensed under LGPLv3. My mentor was kind enough to give me a few test cases to
begin with. As of now client , RpbContent and RpbPutReq have been implemented
as much as required for our porpose. also the test coverage is also good for
all the functions.
A few commands that helped me in debugging are :

+ `make check VERBOSE=1` : used to check all the tests and print the error
    messages on the same screen
+ `export CK_FORK=no` : used before running gdb checks defaults to using a child process to
    run the tests themselves, so it can catch segfaults and the like CK_FORK=no turns that off
+ `tests/check_libriack && gdb tests/check_libriack` to run gdb along with the
    tests, very helpful

Also since I used dynamically allocated object and not static struct for Proto
messages hence i had to use `rpb_content_init` instead of `RPB_CONTENT_INIT` and
likewise to initialise the protobuf-c structure.

Last but not the least `CG_FORK=no valgrind --tool=memcheck
--log-file=riack-tes.log --leak-check=full --show-reachable=yes
tests/check_libriack` used on the tests I wrote gave me the logs of all the
memory leaks that are there in the functions which were all fixed.

Now since both the parts of the project are stable in their own basic state,
next would be to join those to and then make the riak module use this library
and save a few messages in a riak node using the RpbPutReq.



