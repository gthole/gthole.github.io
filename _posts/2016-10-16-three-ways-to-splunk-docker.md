---
layout: post
title:  "Three Ways to Splunk Your Docker Containers"
date:   2016-10-16 09:06:13 -0400
categories: splunk docker
---
Logging from Docker is becoming less of a total pain in the butt.  EnerNOC uses
Splunk Enterprise for its central logging solution, and when we started building
apps with Docker and AWS ECS, getting our logs into Splunk was a real puzzle.

I've set up a couple of different methods to log out to Splunk with varying
success.

### Option 1: Splunk Agent on the Host and Shared Log Volumes
The first thing I tried was provisioning our host ECS machines to run
the Splunk forwarder and share volumes from the running containers.

This was a quick and easy solution since I'd done a lot of Chef-style setup
of the Splunk forwarder before and it wasn't difficult to get going by
having applications in Docker put their logs in a directory that is mounted
to the host.  The Splunk forwarder reads from those logs and sends along.

The drawbacks here:
- It's running a process on the host, which breaks convention of having
  your application stack managed by Docker / ECS.
- The container process is responsible for logging to a file, but if that
  process crashes, the logs don't capture it.

### Option 2: Use the Splunk Log Driver in Docker
With version 1.10, Docker included a `splunk` driver option for its engine.
It took quite some time for AWS ECS to support this option, during which time
I would regularly go to the ECS Agent documentation page to refresh hoping for
an update.

This requires the HTTP Event Collector, which is token based.  When ECS finally
supported it, I updated our task definitions to use the driver and inject
tokens and settings.  I like how easy it is to set metatags like `source` and
`index`, but the big gotcha here is that the logs show up in Splunk as

```
{
    "line": "logger=output foo=bar",
    "source": "stdout",
    "tag": "container-tag"
}
```

This makes simple querying within the `line` attribute quite difficult,
requiring `rex`-style matching to do field extractions or stats.  The
next version of Docker (1.13) will support a `splunk-format` log-driver option
to let the splunk driver send JSON or even just raw strings.  Until then, I
need to find a better way to send the raw events to Splunk.

### Option 3: Use a Splunk Forwarding Container
On instance start (in your Launch Configuration `user_data` script) spin up
a Splunk forwarder container to listen to `syslog` and forward the events to
Splunk.  Then use the `syslog` log driver to send container log messages there.

```
version: '2'
services:
    # Splunk forwarder to send all syslogs to Splunk
    forwarder:
        image: 'splunk/universalforwarder:6.5.0'
        environment:
            SPLUNK_FORWARD_SERVER: 'YOUR_SPLUNK_HOST:9997'
            SPLUNK_FORWARD_SERVER_ARGS: '-method clone'
            SPLUNK_START_ARGS: '--accept-license --answer-yes'
            SPLUNK_ADD: 'udp 1514 -index INDEX_NAME -sourcetype STACK_NAME'
            SPLUNK_USER: 'root'
        ports:
            - '514:1514/udp'
        restart: 'always'

    # Log producer
    app:
        image: 'alpine'
        command: 'sh -c "while true; do echo \"hi `date`\"; sleep 1; done;"'
        logging:
            driver: 'syslog'
            options:
                tag: 'CONTAINER_NAME_OR_TAG'
                syslog-address: 'udp://127.0.0.1'
```

This gets us raw logs, which are more easily searchable until Docker 1.13 is
released.
