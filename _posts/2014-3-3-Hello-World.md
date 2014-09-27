---
layout: post
title: Deploying Sinatra apps using Docker & Ansible
---

I've recently started playing around with Sinatra for simple API based apps. When time came to deploy my app I took the chance to try out a few technologies I've been curious about: Ansible and Docker.

While I've used Ansible for server provisioning, I've never used it for deploys. As Docker goes, this is my first time so bare with me (and take it with a grain of salt, I might be messing up big time!).

I'll try to give a step-by-step guide on how to deploy (and configure a server for a) sinatra app using this technologies. If you just want to find how it felt like, jump to the conclusions!

### The Stack

The application we're setting up consists of:

1. Sinatra backend
2. Postgresql database
3. Client Side MVC (In my case Angular)

We'll be using **unicorn** to serve sinatra requests and **nginx** to route things to unicorn as well as serve our client code and files. 

### Installing docker

*[What is Docker?](https://www.docker.com/whatisdocker/)*

Installing docker is pretty simple and straight forward on a debian based distro:

```
$ sudo apt-get update
$ sudo apt-get install docker.io
$ sudo ln -sf /usr/bin/docker.io /usr/local/bin/docker
$ sudo sed -i '$acomplete -F _docker docker' /etc/bash_completion.d/docker.io
```

[Instalation instructions for other platforms](http://docs.docker.com/installation/)

### The containers

When setting up Docker containers, you should try to keep it [SRP](http://en.wikipedia.org/wiki/Single_responsibility_principle). Meaning we'll have different containers for **Postgres**, **unicorn** (running sinatra) and **nginx**.

Docker provides lot of container images already built, so we'll use them when possible. Docker containers with names in the following format "package_name" (ie: "postgres") are maintained by the Docker team, while containers name "user_name#package_name" are community containers.

Let's download the containers we'll need:

```
$ sudo docker pull postgres
$ sudo docker pull nginx
$ sudo docker pull ruby
```

#### Postgresql

The container provided with Docker comes configured and fully functional, if you want to test it run:

```
$ docker run --name db -d postgres
```

This command starts a deamonized container running postgres with the port 5432 exposed. You can connect to the database by either running:

```
$ psql -h localhost -U postgres # make sure you don't have other programs on listening on 5432
```

or by firing up another container:

```
$ docker run -it --link db:postgres --rm postgres sh -c 'exec psql -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres' # the --rm flag tells docker to destroy the container after the command finishes (when you exit psql)
```

#### nginx
#### ruby

