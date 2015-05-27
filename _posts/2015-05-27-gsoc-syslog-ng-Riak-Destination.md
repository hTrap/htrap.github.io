---
layout: post
title: Syslog-ng project - Riak destination (Initial Steps)
tags: gsoc community google bonding syslog-ng 
category: work self
comments: true
authon: Parth Oberoi
meta: Work So Far
---

Having been selected for GSoC under **The Syslog-ng Project**  for
the Project *Riak Destination* i.e Adding a new destination module to Syslog-ng,
my first task was to set up a template for the configuration to be used by
syslog-ng users using riak as a destination.The syslog-ng application is
configured by editing the syslog-ng.conf file.

The syslog-ng.conf mainly consists of three parts i.e `source`, `destination`
and `log`, a very simple configuration for syslog-ng will be 

```
@version:3.7

source s_name { internal(); };

destination d_name { file("/var/log/messages_syslog-ng.log"); };

log { source(s_name); destination(d_name);
```

Writing the configuration of the riak module first was beneficial as being a 
top-down approach we could decide on the templatable options that the users
would like to edit to fit their own needs. Below is the configuration that 
would be used as the basis for further development:

```

@version: 3.7

source s_demo {
    system();
    intenal();
};

destination d_riak {
    riak{
        host("localhost")
        port(8087)
        bucket("logs_${YEAR}${MONTH}${DAY}" type("message") mode("store"))
        key("${UNIXTIME}-${UUID)")
        value("${format-json --scope selected-macros}")
        };
};

log {
    source(s_demo); destination(d_riak);
};
```

The two interesting options in this config are `bucket` and `key`. The bucket 
option further uses two sub-options(since both these options related to the bucket)
`type` and `mode`. `type` takes in the bucket-type being used and `mode` is set to 
`set` or `store` which describes the mode we are using i.e. storing messages in a 
bucket under specific keys or using a set which is a riak feature (a combo of retrieve 
and store).The bucket name and the key is templateable.


The next part was to parse this configuration, for which BNF form of grammar had to be written that parses the options used in the initial configuration and passes that option through a function that saves it the RiakDestDriver struct. for eg:
`server("localhost")` - option used in the config file,
This option get parsed by the grammar snippet below

```
riak_option
        : KW_HOST '(' string ')'
          {
            riak_dd_set_host(last_driver, $3);
            free($3);
          }
```
This sets the host by calling `riak_dd_set_host(last_driver, $3);`
which is defined below -

```c
void
riak_dd_set_host(LogDriver *d, char *host)
{
  RiakDestDriver *self = (RiakDestDriver *)d;

  free(self->host);
  self->host = strdup(host);
}
```
Similiar grammar and functions to save the options is to be written for all the options
in the configuration.