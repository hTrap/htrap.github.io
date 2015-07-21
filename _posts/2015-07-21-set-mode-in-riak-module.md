---
layout: post
title: Adding SET mode with bulk functionality in the Syslog-ng Riak module
tags: gsoc riak Syslog-ng destination libriack 
category: work self gsoc
comments: true
author: Parth Oberoi
meta: Syslog-ng Riak set mode 
---

As suggested in my last
[post](http://thetechtrap.com/posts/gsoc-syslog-ng-libriak-connection-destination/),
the next step was to add a `set` mode to store the log messages in the sets. 
To accomplish that I wrapped the `DtUpdateReq` protocol which uses to Riak CRDT
to store values in a set.

The `DtUpdateReq` [(read more about
it)](http://docs.basho.com/riak/latest/dev/references/protocol-buffers/dt-store/) 
takes in bucket name, key, the bucket-type to be used which in this case is
`set` , also it takes in the DtOp protocol message or the Data Type operation
which in our case is the SetOp protocol.
The SetOp protocol is the heart of the set mode, it has a repeated field of
`adds` to add a message in a set and `removes` to remove the message from the
set. The repeated fields helps us to add or remove any number of messages in a
single request i.e in bulk.

To add a single message in a set `RIACK_SETOP_FIELD_ADD` flag and to
remove a message from a set `RIACK_SETOP_FIELD_REMOVE` is used in the
`riack_setop_set` function 
### For the bulk addition :-

+ flush_lines(NUM) is added to the configuration file, which works under the
    set mode only. This takes in NUM which is the number of messages to be
    stored in the SetOp repeated adds field buffer before sending it to the
    client.

+ The `worker_insert_result_t` function calls the `riack_setop_set` function
    NUM number of times as below:-

    ```
    if (self->flush_lines) {
      for (idx=0;idx<self->flush_lines;idx++) {
        riack_setop_set (setop,
        RIACK_SETOP_FIELD_BULK_ADD,
        idx,
        value_res,
        RIACK_SETOP_FIELD_NONE);
        }
    }
    ```

+ The `riack_setop_set` fuction checks if the index(idx) is 0 and it allocates
    memory using malloc and adds the message to buffer, and if the index is not
    zero thoen it reallocates the memory to `sizeof(ProtobufCBinaryData) *
    (idx+1)` and adds the message to buffer,
    also `setop->n_adds` is incremented in every call.

###The main problems I faced :-

+ I recieved a `errno=3` on using sets to save data in riak, The problem was, in
    order to use the CRDT set you must first
    [create](http://docs.basho.com/riak/latest/dev/using/data-types/#Setting-Up-Buckets-to-Use-Riak-Data-Types) 
    a bucket type that sets the datatype bucket parameter set.
    Fixed it by using

    ```
    riak-admin bucket-type create sets '{"props":{"datatype":"set"}}' 
    riak-admin bucket-type activate sets
    ```

+ Then was the `double free or corruption (out)` error which was caused when
    freeing the setop pointer while using the bulk mode. The problem was that initially I was using `sizeof(char
    *)` to allocate memory which worked because of luck but  setop->adds is an
    array of ProtobufCBinaryData elements, so the base size I had  to use is
    `sizeof(ProtobufCBinaryData)` which fixed that error.

+ To ease development i.e. not having to compile libriack, then relink
    syslog-ng, etc, the libriack library was added as a part of syslog-ng riak
    module it also makes easier for distributions, because they don't
    have to package libriack separately, it being a part a syslog-ng.

