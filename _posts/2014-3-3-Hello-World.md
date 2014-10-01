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

This layer will run sinatra through unicorn and connect to the db container. To setup unicorn I've created a *unicorn.conf* file inside the root of my sinatra app.

```
worker_processes 8
working_directory "/home/rubier/YOUR_APP/" # Change this to your app name

listen 'localhost:8888', :backlog => 512   # We'll listen on a ip socket instead of a file,
                                           # to facilitate communication with the nginx container.
timeout 120
pid "/home/rubier/unicorn.pid"             # This part is actually not very important since we'll
                                           # destroy the container on every new deploy.

preload_app true
if GC.respond_to?(:copy_on_write_friendly=)
  GC.copy_on_write_friendly = true
end

before_fork do |server, worker|
 old_pid = "#{server.config[:pid]}.oldbin"
 if File.exists?(old_pid) && server.pid != old_pid
   begin
     Process.kill("QUIT", File.read(old_pid).to_i)
   rescue Errno::ENOENT, Errno::ESRCH
     # someone else did our job for us
   end
 end
end
```

Unicorn is a rack server, therefore we'll also need a simple *config.ru* file:

```
require 'sinatra'
set :env, :production
disable :run
require 'app.rb' # Of whatever you've called your application file
run Sinatra::Application
```

Now that the configuration for unicorn and rack is done, let's create a custom container for our app. Inside your sinatra app create the following *Dockerfile*:

```
FROM ubuntu:14.04
MAINTAINER Your Name <your@email.address>

RUN apt-get update && apt-get install -y curl procps && rm -rf /var/lib/apt/lists/*

# Install rbenv and some dependencies we'll need for ruby and nokogiri
RUN apt-get update \
  && apt-get install rbenv -y \
  && apt-get install git -y \
  && apt-get install build-essential -y \
  && apt-get install autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev -y
RUN apt-get update && apt-get install libxslt-dev libxml2-dev -y && apt-get install libpq-dev -y

# Create an unprivileged user
RUN adduser rubier
USER rubier

# skip installing gem documentation
RUN echo 'gem: --no-rdoc --no-ri' >> ~/.gemrc

# Install ruby-build
RUN git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
# Compile ruby and install bundler
RUN rbenv install 2.1.3 && rbenv global 2.1.3 && rbenv rehash
RUN rbenv exec gem install bundler

# Copy code from the current directory to the docker container
COPY . /app/

# Make an user privileged copy of the code
RUN cp -R /app /home/rubier/
# Run bundle install
RUN cd /home/rubier/app && rbenv exec bundle install

# Expose unicorn socket
EXPOSE 8888
WORKDIR /home/rubier/tephi

# Start unicorn when the container initializes
CMD rbenv exec bundle exec unicorn -c ./unicorn.conf
```

Start the container:

```
sudo docker run -d --name app --link db:postgres -p 8000:8888 eidge/ruby-tesseract-imagemagick
```



