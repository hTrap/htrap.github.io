---
layout: post
title: Using persistent storage for set mode in bulk and fixing memory leaks
tags: gsoc riak Syslog-ng 
category: work self gsoc
comments: true
author: Parth Oberoi
meta: Syslog-ng Riak set mode 
---

After implementation of the bulk function, the next task was to clean up my
code in accordance to syslog-ng standards. To acheive that the following
standards were taken care of throughout the code I had written so far :

+ Indentation should be done only with spaces,
+ Each indentation level should be of two spaces,
+ '{' is indented with 2 spaces inside from the preceding line,
+ No space before '(' unless it is after an `if`, `switch`, `=` or `while`
+ For any other doubts I looked at the existing syslog-ng code which used the
syntax in doubt

After this I realised that the code used for calling `riack_setop_set`
(`flush_lines` times) during the set bulk mode

```
if (self->flush_lines)
  {
    for (idx=0;idx<self->flush_lines;idx++) 
      {
        riack_setop_set (setop,
          RIACK_SETOP_FIELD_BULK_ADD,
          idx,
          value_res,
          RIACK_SETOP_FIELD_NONE);
      }
  }
```

used to re-initialize after every log-message hence the bulk mode was failing
to work.
The problem was, `riak_worker_insert` function is called every time a log
message is received, so the index value(idx) lasted as long as the message
lasted and then was again reinitialized to 0 which wasn't requried.I
wanted the `flush_lines`, idx(now `flush_index`) and `setop`(where we
store the bulk messages in buffer) to last until the `flush_index` becomes equal
to the `flush_lines` and then push it to the riak node. Anything that has to live
longer than a single call to `riak_worker_insert()`, everything you want to
persist from one message to another, has to live in RiakDestDriver's structure
which is a sort of a persistent storage hence `flush_index`, `flush_lines` and
`setop` were added into the `RiakDestDriver`'s structure. 

Now to check for the memory leaks, I used valgrind in the following steps:-

+ used `G_SLICE=always-malloc valgrind --tool=memcheck --leak-check=full --verbose
--log-file=syslog-ng.valgrind.log
--suppressions=$HOME/syslog-gsoc/syslog-ng/contrib/valgrind/syslog-ng.supp
sbin/syslog-ng -Fvde -f etc/syslog-ng.conf` in my installation path of syslog-ng i.e install_new/syslog-ng.
+ After issuing the command I used a simple script `for i in {1..5000}; do echo
commandlogtesttext$i | nc 127.0.0.1 12345; done` to send messages to riak in
set bulk mode.

+ This created a syslog-ng.valgrind.log file with showed all the major leaks
that had to be fixed.

Also earlier the `riack_client_connect()` function was being called every time
just for a check, instead of that I added a function
`riack_client_is_connected()` to the library which returned 1(connected) if `client->fd >= 0` 
else 0(not connected) if `client->fd == -1` also I did set `client->fd =>-1` in `riack_client_new()`and `riack_client_disconnect` so that client->conn
became redundant and be removed.
