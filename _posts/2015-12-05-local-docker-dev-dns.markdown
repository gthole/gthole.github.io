---
layout: post
title:  "DNS for Local Docker Development"
date:   2015-12-05 09:03:01 -0500
categories: docker mdns avahi
---
With Docker for Mac, the Docker team removed a pretty hefty load of frustration
from developers - no more `docker-machine` and setting environment variables
for `docker` tcp communication.

There are drawbacks though - the big one for me is separation of project spaces.
`docker-machine` lets a developer logically group containers into a VM that can
be reset or destroyed without affecting the others.  Container ports won't collide
between projects, and you can bind DNS for a given project to be able to refer
to it locally.

That last one is important to me.  When I spin up a web application locally,
I don't want to have to run `docker-machine ip vm-name` to find out where to
point my browser or API client. (Plus, a local IPV4 address isn't very descriptive.)
I could edit my `/etc/hosts` file, but if the machine's IP changes then I have
to edit it.

The solution is to spin up a small Avahi container alongside my web
application with `docker-compose`.  [Avahi](avahi.org) is a Linux implementation
of multi-casting DNS, which is shipped on Mac OSX as [Bonjour](https://support.apple.com/bonjour).
This lets you use `<VM-name>.local` as the DNS entry for the VM on your local
network.

So, for example, with a basic wordpress setup:

```yaml
version: '2'
services:
    avahi:
        image: 'enernoclabs/avahi:latest'
        logging:
            driver: 'none'
        network_mode: 'host'
    mysql:
        image: 'mariadb'
        environment:
            MYSQL_ROOT_PASSWORD: 'foobarbazzhands'
    app:
        image: 'wordpress'
        ports:
            - '8080:80'
        environment:
            WORDPRESS_DB_PASSWORD: 'foobarbazzhands'
```

Put the above in your `docker-compose.yml`, then:

```bash
# Start the VM
$ docker-machine create -d virtualbox wordpress

# Set your environment variables
$ eval "$(docker-machine env wordpress)"

# Start the containers
$ docker-compose up
```

Then in your browser, go to
[http://wordpress.local:8080](http://wordpress.local:8080) to see your running
Wordpress app!
