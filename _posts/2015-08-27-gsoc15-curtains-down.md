---
layout: post
title: GSoC15 Curtains Down 
tags: gsoc riak Syslog-ng 
category: work self gsoc
comments: true
author: Parth Oberoi
meta: Syslog-ng Riak set mode 
---
##Setting Up the Riak destination
To set up a riak destination you must first download and install the following
:

- [Erlang](http://docs.basho.com/riak/latest/ops/building/installing/erlang/)
- [Riak](http://docs.basho.com/riak/latest/downloads/)
- [GoogleProtocol
Buffers](https://developers.google.com/protocol-buffers/docs/downloads)
-   [Protobuf-c](https://github.com/protobuf-c/protobuf-c)

and since it is a destination for Syslog-ng , you ust have that installed.


Then you have to clone the Riak repo:

`git clone https://github.com/hTrap/syslog-ng.git --branch f/riaktest`
and follow [this wiki](https://github.com/hTrap/syslog-ng/wiki/Riak-Destination)

##Testing the set up
Here is the Syslog-ng configuration that we can use for testing the destination:

```
@version: 3.7

source s_demo {
    tcp(ip("127.0.0.1") port(12345) flags(no-parse));
};

destination d_riak {
    riak(
        host("localhost")
        port(8087)
        bucket("logs_${YEAR}${MONTH}${DAY}" type("message") mode("store"))
        key("${UNIXTIME}-${UUID}")
        value("$(format-json --scope selected-macros)" charset("utf8") content_type("application/json"))
        );
};

log {
    source(s_demo); destination(d_riak);
};
```
This takes in a message from the ip and port defined in the source and saves it into riak node running locally in its store mode.

Then you can simply take the following steps:

- Ensure that Riak is running by `riak start`
- Run `syslog-ng -Fvde`
- open another terminal and issue a message targetted to the above port `echo YOURMESSAGE | nc 127.0.0.1 12345`
- Now check on the previous terminal if riak did receive that message.

You can also check the bulk(set) feature by changing the `mode("store")` to `mode("set")` in the above configuration, and add `flush_lines(100)` where 100 is the number of lines you want to set in bulk i.e once 100 messages are received, they are ent to the Riak Node.
Your  configuration will now look like-

```
@version: 3.7

source s_demo {
    tcp(ip("127.0.0.1") port(12345) flags(no-parse));
};

destination d_riak {
    riak(
        host("localhost")
        port(8087)
        bucket("logs_${YEAR}${MONTH}${DAY}" type("message") mode("set"))
        key("${UNIXTIME}-${UUID}")
        value("$(format-json --scope selected-macros)" charset("utf8") content_type("application/json"))
        flush_lines(100)
        );
};

log {
    source(s_demo); destination(d_riak);
};
```
Then restart `syslog-ng -Fvde` and then open another terminal and then run
`for i in {1..2000}; do echo logtesttext$i | nc 127.0.0.1 12345; done`.
This would send 2000 messaages to the port which would be saved in riak in a bulk set of 100 specified earlier in the configuration.
So you must now receive 20 acknowledgments in the syslog-ng terminal.

If you encounter something unusual, do report it to me on htrapdev@gmail.com



##My experience

This being my first GSoC, I was really excited as well as scared.
Excited because, _well its gsoc_ and scared because building an entire project in C can be a __segmentation fault studded nightmare__. 
But as my project progressed I was exposed to tools like gdb and valgrind and also my command over the language grew. 
For sure I had to be persistent to proceed as a small bug near the end of the project consumed an entire week 
and later looked so petty. But now the GSoC period is over, and I am still excited and still want to contribute more. GSoC gave me the 
exposure I needed to the open source world.

My mentor for the project Gergely Nagy aka algernon was really better than the best mentor I could have imagined. 
He was very welcoming, he guided me wherever I got stuck,
 gave me a few pointers and he was usually a ping away on irc. I cannot thank him enough for his mentorship.
 
##Future plans for the destination
I would really love to continue improving on the destination and have a few plans in mind like :-

- wrapping more riak protocols in C
- adding it to a separate library Riack for independent usage
- more will be added here soon

