---
layout: post
title: Connecting Riak module with the Riack library
tags: gsoc riak Syslog-ng destination libriack 
category: work self gsoc
comments: true
author: Parth Oberoi
meta: Protocol Buffers
---

As suggested in my last
[post](http://thetechtrap.com/posts/gsoc-syslog-ng-riak-client-library/),
the next step was to connect the Riak module to use the
[Riack](https://github.com/algernon/riack) library and then use it to save
messages and logs in the Riak node using RpbPutReq protocol. In order to accomplish this
the following tasks were done :
###Changes for Syslog-ng Riak module

+ Adding automake  commands to `configure.ac` and `Makefile.am` files so that the module
    recognizes the presence of the riack library in the prefix location by using
    `./configure CFLAGS="-ggdb3 -O0" --prefix=$HOME/install_new/syslog-ng
    --disable-java --with-libriack=$HOME/install_new/riack` .
    The major problem I faced here was to find the libraries in the installed
    location. Initially I used the absolute location of my installation path of
    riack but that wouldn't work on all the systems so then I used the
    following in the `modules/riak/Makefile.am` (as you can see its a diff so + is add line and - is remove
    line) which solved the issue.

    ```
    modules_riak_libriak_la_LIBADD   =   \
    -   $(MODULE_DEPS_LIBS)         \
    -   -L/${HOME}/install_new/riack/lib      \
    -   -lriack
    +   $(MODULE_DEPS_LIBS) $(RIACK_LIBS)
    ```

+ Now since the library was connected the next task was to use the library to
    store messages in the riak node. This was done in the
    `riak_worker_insert()`
    function that does all the work i.e. using `log_template_format(...
    ,result)` it converts the template used in the configuration to a string
    result for eg. our example
    [configuration](https://github.com/hTrap/syslog-ng/blob/f/riak/modules/riak/riak-example.conf)
    consists of a template bucket `"logs_${YEAR}${MONTH}${DAY}"` which is
    converted to `'logs_20150626'` and saved in `result-str` using the
    `log_template_format` which is then used
    later and sent to the riak node.

+ Then the Riack library functions were used to send the messages to riack node
    for the *store* mode using RpbPutReq protocol.

###Riack library / libriack

The major task for the library was to package the data specific to riak
guidelines and then send it.
A riack request that is sent should be encoded in a specific way i.e.
Each operation consists of a request message and one or more response messages.
Messages are all encoded the same way, consisting of:

+ 32-bit length of message code + Protocol Buffers message in network order
+ 8-bit message code to identify the Protocol Buffers message
+ N bytes of Protocol Buffers-encoded message
The message codes are for each operation are listed
[here](http://docs.basho.com/riak/latest/dev/references/protocol-buffers/#Message-Codes)

This way of serialization of the riak message was abstracted away from the
`riack_client_send` method so that all types of operations can use this function
to send operation requests.
That last step was to recieve a response from the riak node which shows that
the message has been saved in the node. The `riack_client_recv()` however still receives
raw bytes and still has to be further developed for deconstructing the raw
bytes received into something sensible.

Changes were also made in the library so that Syslog-ng does not crash when
`charset` and `content_type(initially ctype)` are not specified for a value in the
configuration. This was done using the `has_type` member of the optional
message protocol which was set to 0 if the above were not present and to 1 if
they were present.


At this point the riak module is successfully using the Riack library to save
the messages/logs into the riak node in the `store` mode while using
`RpbPutReq` protocol.
Next step which I am currently working on is to store messages in riak using  the `set`
mode by using `DtUpdateReq` protocol.
