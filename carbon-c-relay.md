carbon-c-relay(1) -- graphite relay, aggregator and rewriter
============================================================

## SYNOPSIS

`carbon-c-relay` `[-hvdmstD]` `-f` <config-file> `[ options` ... `]`

## DESCRIPTION

**carbon-c-relay** accepts, cleanses, matches, rewrites, forwards and
aggregates graphite metrics by listening for incoming connections and
relaying the messages to other servers defined in its configuration.
The core functionality is to route messages via flexible rules to the
desired destinations.

## OPTIONS

These options control the behaviour of `carbon-c-relay`.

  * `-v`:
    Print version string and exit.

  * `-d`:
    Enable debug mode, this prints statistics to stdout and prints extra
    messages about some situations encountered by the relay that normally
    would be too verbose to be enabled.  When combined with `-t`
    (test mode) this also prints stub routes and consistent-hash ring
    contents.

  * `-m`:
    Change statistics submission to be like carbon-cache.py, e.g. not
    cumulative.  After each submission, all counters are reset to `0`.

  * `-s`:
    Enable submission mode.  In this mode, internal statistics are not
    generated.  Instead, queue pressure and metrics drops are reported on
    <stdout>.  This mode is useful when used as submission relay which' job is
    just to forward to (a set of) main relays.  Statistics about the
    submission relays in this case are not needed, and could easily cause a
    non-desired flood of metrics e.g. when used on each and every host
    locally.

  * `-t`:
    Test mode.  This mode doesn't do any routing at all, but instead reads
    input from stdin and prints what actions would be taken given the loaded
    configuration.  This mode is very useful for testing relay routes for
    regular expression syntax etc.  It also allows to give insight on how
    routing is applied in complex configurations, for it shows rewrites and
    aggregates taking place as well.

  * `-D`:
    Deamonise into the background after startup.  This option requires
    `-l` and `-P` flags to be set as well.

  * `-f` <config-file>:
    Read configuration from <config-file>.  A configuration consists of
    clusters and routes.  See [CONFIGURATION SYNTAX][] for more
    information on the options and syntax of this file.

  * `-i` <iface>:
    Open up connections on interface <iface>.  Currently only one
    interface can be specified, and it is specified by its IP address, not
    the interface name.  By default, the relay opens listeners on all
    available interfaces (a.k.a. `0.0.0.0`).

  * `-l` <log-file>:
    Use <log-file> for writing messages.  Without this option, the relay
    writes both to <stdout> and <stderr>.  When logging to file, all
    messages are prefixed with `MSG` when they were sent to <stdout>, and
    `ERR` when they were sent to <stderr>.

  * `-p` <port>:
    Listen for connections on port <port>.  The port number is used for
    both `TCP`, `UDP` and `UNIX sockets`.  In the latter case, the socket
    file contains the port number.  The port defaults to `2003`, which is
    also used by the original `carbon-cache.py`.

  * `-w` <workers>:
    Use <workers> number of threads.  The default number of workers is
    equal to the amount of detected CPU cores.  It makes sense to reduce
    this number on many-core machines, or when the traffic is low.

  * `-b` <batchsize>:
    Set the amount of metrics that sent to remote servers at once to
    <batchsize>.  When the relay sends metrics to servers, it will
    retrieve <batchsize> metrics from the pending queue of metrics waiting
    for that server and send those one by one.  The size of the batch will
    have minimal impact on sending performance, but it controls the amount
    of lock-contention on the queue.  The default is `2500`.

  * `-q` <queuesize>:
    Each server from the configuration where the relay will send metrics
    to, has a queue associated with it.  This queue allows for disruptions
    and bursts to be handled.  The size of this queue will be set to
    <queuesize> which allows for that amount of metrics to be stored in
    the queue before it overflows, and the relay starts dropping metrics.
    The larger the queue, more metrics can be absorbed, but also more
    memory will be used by the relay.  The default queue size is `25000`.

  * `-L` <stalls>:
    Sets the max mount of stalls to <stalls> before the relay starts
    dropping metrics for a server.  When a queue fills up, the relay uses
    a mechanism called stalling to signal the client (writing to the
    relay) of this event.  In particular when the client sends a large
    amount of metrics in very short time (burst), stalling can help to
    avoid dropping metrics, since the client just needs to slow down for a
    bit, which in many cases is possible (e.g. when catting a file with
    `nc`(1)).  However, this behaviour can also obstruct, artificially
    stalling writers which cannot stop that easily.  For this the stalls
    can be set from `0` to `15`, where each stall can take around 1 second
    on the client.  The default value is set to `4`, which is aimed at the
    occasional disruption scenario and max effort to not loose metrics
    with moderate slowing down of clients.

  * `-S` <interval>:
    Set the interval in which statistics are being generated and sent by
    the relay to <interval> seconds.  The default is `60`.

  * `-B` <backlog>:
    Sets TCP connection listen backlog to <backlog> connections.  The
    default value is `3` but on servers which receive many concurrent
    connections, this setting likely needs to be increased to avoid
    connection refused errors on the clients.

  * `-I` <timeout>:
    Specifies the IO timeout in milliseconds used for server connections.
    The default is `600` milliseconds, but may need increasing when WAN
    links are used for target servers.  A relatively low value for
    connection timeout allows the relay to quickly establish a server is
    unreachable, and as such failover strategies to kick in before the
    queue runs high.

  * `-c` <chars>:
    Defines the characters that are next to `[A-Za-z0-9]` allowed in
    metrics to <chars>.  Any character not in this list, is replaced by
    the relay with `\_` (underscore).  The default list of allowed
    characters is `-\_:#`.

  * `-H` <hostname>:
    Override hostname determined by a call to `gethostname`(3) with
    <hostname>.  The hostname is used mainly in the statistics metrics
    `carbon.relays.`<hostname>`.`<...> send by the relay.

  * `-P` <pidfile>:
    Write the pid of the relay process to a file called <pidfile>.  This
    is in particular useful when daemonised in combination with init
    managers.

## CONFIGURATION SYNTAX

The config file supports the following syntax, where comments start with
a `#` character and can appear at any position on a line and suppress
input until the end of that line:

```
cluster <name>
    <forward | any_of [useall] | failover |
	<carbon_ch | fnv1a_ch | jump_fnv1a_ch> [replication <count>]>
        <host[:port][=instance] [proto <udp | tcp>]> ...
    ;

cluster <name>
    file [ip]
        </path/to/file> ...
    ;

match
        <* | expression ...>
    send to <cluster ... | blackhole>
    [stop]
    ;

rewrite <expression>
    into <replacement>
    ;

aggregate
        <expression> ...
    every <interval> seconds
    expire after <expiration> seconds
    [timestamp at <start | middle | end> of bucket]
    compute <sum | count | max | min | average |
             median | percentile<%> | variance | stddev> write to
        <metric>
    [compute ...]
    [send to <cluster ...>]
    [stop]
    ;

send statistics to <cluster ...>
    [stop]
    ;

include </path/to/file/or/glob>
    ;
```

A simple example that would send all incoming metrics to a cluster of
three servers could look like:

```
cluster ams4   # you can choose any other name here
   jump_fnv1a_ch replication 2
      10.0.0.1:2003
      10.0.0.2:2003
      10.0.0.3:2003
   ;

match *
   send to ams4
   stop;
```

For more documentation about the available clusters and routes, please
review the [README file][readme] on GitHub.

[readme]: https://github.com/grobian/carbon-c-relay/blob/master/README.md
          "README file"

## BUGS

Please report them at:
<https://github.com/grobian/carbon-c-relay/issues>

## AUTHOR

Fabian Groffen &lt;grobian@gentoo.org&gt;

## SEE ALSO

All other utilities from the graphite stack.