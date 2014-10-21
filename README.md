docker-gen
=====

`docker-gen` is a file generator that renders templates using docker container meta-data.

It can be used to generate various kinds of files for:

 * **Centralized logging** - [fluentd](https://github.com/jwilder/docker-gen/blob/master/templates/fluentd.conf.tmpl), logstash or other centralized logging tools that tail the containers JSON log file or files within the container.
 * **Log Rotation** - [logrotate](https://github.com/jwilder/docker-gen/blob/master/templates/logrotate.tmpl) files to rotate container JSON log files
 * **Reverse Proxy Configs** - [nginx](https://github.com/jwilder/docker-gen/blob/master/templates/nginx.tmpl), [haproxy](https://github.com/jwilder/docker-discover), etc. reverse proxy configs to route requests from the host to containers
 * **Service Discovery** - Scripts (python, bash, etc..) to register containers within [etcd](https://github.com/jwilder/docker-register), hipache, etc..

===

### Installation

There are three common ways to run docker-gen:
* on the host
* bundled in a container with another application
* separate standalone containers

#### Host Install

Linux binaries for release [0.3.4](https://github.com/jwilder/docker-gen/releases)

* [amd64](https://github.com/jwilder/docker-gen/releases/download/0.3.4/docker-gen-linux-amd64-0.3.4.tar.gz)
* [i386](https://github.com/jwilder/docker-gen/releases/download/0.3.4/docker-gen-linux-i386-0.3.4.tar.gz)

Download the version you need, untar, and install to your PATH.

```
$ wget https://github.com/jwilder/docker-gen/releases/download/0.3.4/docker-gen-linux-amd64-0.3.4.tar.gz
$ tar xvzf docker-gen-linux-amd64-0.3.4.tar.gz
$ ./docker-gen
```

#### Bundled Container Install

Docker-gen can be bundled inside of a container along-side and applications.

[jwilder/nginx-proxy](https://index.docker.io/u/jwilder/nginx-proxy/) trusted build is an example of
running docker-gen within a container along-side nginx.
[jwilder/docker-register](https://github.com/jwilder/docker-register) is an example or running
docker-gen within a container to do service registration with etcd.

#### Separate Container Install

It can also be run as two separate containers using the [jwilder/docker-gen](https://index.docker.io/u/jwilder/docker-gen/)
image virtually any other image.

This is how you could run the official [nginx](https://registry.hub.docker.com/_/nginx/) image and
have dockgen-gen generate a reverse proxy config in the same way that `nginx-proxy` works.  You may want to do
this to prevent having the docker socket bound to an publicly exposed container service.

Start nginx with a shared volume:

```
$ docker run -d -p 80:80 --name nginx -v /tmp/nginx:/etc/nginx/conf.d -t nginx
```

Start the docker-gen container with the shared volume:
```
$ docker run --volumes-from nginx \
    -v /var/run/docker.sock:/tmp/docker.sock \
   -v $(pwd)/templates:/etc/docker-gen/templates
   -t docker-gen -notify-sighup nginx -watch --only-published /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

===

### Usage
```
$ docker-gen
Usage: docker-gen [-config file] [-watch=false] [-notify="restart xyz"] [-notify-sighup="nginx-proxy"] [-interval=0] [-endpoint tcp|unix://..] <template> [<dest>]
```

*Options:*

```
  -config="": Use the specified config file instead of command-line options.  Multiple templates can be defined and
              they will be executed in the order that they appear in the config file.
  -endpoint="": docker api endpoint [tcp|unix://..]. This can also be set w/ a `DOCKER_HOST` environment.
  -interval=0:run notify command interval (s). Useful for service registration use cases.
  -notify="": run command after template is regenerated ["restart xyz"]. Useful for restarting nginx,
              reloading haproxy, etc..
  -notify-sighup="": send HUP signal to container.  Equivalent to `docker kill -s HUP container-ID`
  -only-exposed=false: only include containers with exposed ports
  -only-published=false: only include containers with published ports (implies -only-exposed)
  -version=false: show version
  -watch=false: run continuously and monitors docker container events.  When containers are started
                or stopped, the template is regenerated.
```

If no `<dest>` file is specified, the output is sent to stdout.  Mainly useful for debugging.

===

### Examples

* [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
* [Docker Log Management With Fluentd](http://jasonwilder.com/blog/2014/03/17/docker-log-management-using-fluentd/)
* [Docker Service Discovery Using Etcd and Haproxy](http://jasonwilder.com/blog/2014/07/15/docker-service-discovery/)

#### NGINX Reverse Proxy Config

[jwilder/nginx-proxy](https://index.docker.io/u/jwilder/nginx-proxy/) trusted build.

Start nginx-proxy:

```
$ docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock -t jwilder/nginx-proxy
```

Then start containers with a VIRTUAL_HOST env variable:

```
$ docker run -e VIRTUAL_HOST=foo.bar.com -t ...
```

If you wanted to run docker-gen directly on the host, you could do it with:

```
$ docker-gen -only-published -watch -notify "/etc/init.d/nginx reload" templates/nginx.tmpl /etc/nginx/sites-enabled/default
```

#### Fluentd Log Management

This template generate a fluentd.conf file used by fluentd.  It would then ships log files off
the host.

```
$ docker-gen -watch -notify "restart fluentd" templates/fluentd.tmpl /etc/fluent/fluent.conf
```

#### Service Discovery in Etcd


This template is an example of generating a script that is then executed.  This tempalte generates
a python script that is then executed which register containers in Etcd using it's HTTP API.

```
$ docker-gen -notify "/bin/bash /tmp/etcd.sh" -interval 10 templates/etcd.tmpl /tmp/etcd.sh
```


### Development

This project uses [glock](https://github.com/robfig/glock) for managing 3rd party dependencies.
You'll need to install glock into your workspace before hacking on docker-gen.

```
$ git clone <your fork>
$ glock sync github.com/jwilder/docker-gen
$ make
```

### TODO

 * Add event status for handling start and stop events differently
 * Add a way to filter out containers in templates
