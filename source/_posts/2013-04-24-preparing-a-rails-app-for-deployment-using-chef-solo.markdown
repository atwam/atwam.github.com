---
layout: post
title: "From pow to a deployed rails app using chef, capistrano and vagrant - Part 1"
date: 2013-04-24 22:09
comments: true
published: false
categories: [rails, chef, vagrant, capistrano]
---

So, let's be honest, I've been quite lazy here. My latest rails application was deployed with Heroku. Lots of fun, a very pleasant experience.
But now I have this new shinny application that I don't want to deploy using Heroku :

* Because I am worried that if I start with Heroku, I'll be too lazy to switch later.
* Because I know that at some point, my application is going to need plugins and binaries that I can't get on Heroku
* Because I'd rather have a portable application that I can deploy easily on any type of server.

## 1. Planning and strategy
I have this nice application, let's call it cube. So far it has lived very happily in rails dev environment, run only by the marvelous [pow](http://pow.cx).

My goal is to have :

* A server configuration using chef
* An easy deployment using capistrano
* A way to localy run a copy of the server deployment using vagrant

This should allow me to easily change/add services on my box, and scale it up should the need arise.

## 2. Chef in the kitchen

[Chef](http://www.opscode.com/chef/) will be used to handle the configuration/bootstrapping of my servers.
I've done my fair share of playing with ssh in the past, only to realize that administering a server for a side project using ssh is fun until
you have to come back to it after a few months and remember where all the things are and where you should do you change.

With chef, the configuration is stored on git, and if I need to change anything, there's only one place to go.
Soooo, let's start.

```bash
$ mkdir chef-repo && cd chef-repo
$ cat > Gemfile <<EOF
source :rubygems

gem "chef" # The main gem
gem "knife-solo" # Chef solo to bootstrap/configure chef on individual machines through ssh
gem "berkshelf" # To manage cookbooks
EOF
$ bundle install
# If you see an error requesting gecode when you run `bundle install`
# then `brew install --use-llvm gecode` should be good enough to make it work.
</small>
```

That should cover the installation of the main `chef` gem and of `knife-solo`. Chef can be configured to use with a chef server (useful to manage many machines)
but for now I'll be using `knife-solo` (also known as chef-solo) which should be good enough.


Now let's initiate a chef repository in my directory. This repository will hold all the configuration for the server(s) I am going to deploy to.

```bash
$ knife solo init . # Creates the chef-repo here
```

If you have read a bit about chef (of course you have), you know that one of the main ingredients (ahah) are the *cookbooks*.
Opscode has gathered many cookbooks contributed by the community.
The standard way of installing these cookbooks is through `knife cookbook site install blah`, but I have included the `berkshelf` gem which will handle
our `cookbooks` directory for us and get all the proper cookbooks and their dependencies. The list of cookbooks is managed in a `Berksfile`.

For now, let's start by installing some cookbooks. We'll see later to configure them and our app.

```ruby Berksfile
site :opscode

cookbook 'apt'
cookbook 'build-essentials'
cookbook 'nginx'
cookbook 'database'
```
A few comments on these cookbooks ..

apt
: Used to make sure this package manager is installed on the ubuntu box that we'll be using.

build-essentials
: Make sure we can build stuff. This one will be included anyway since it's a dependency of so many other packages.

[nginx](http://community.opscode.com/cookbooks/nginx)
: Planning to use this lightweight web server to serve our rails app

[database](http://community.opscode.com/cookbooks/database)
: This cookbook is an alternative to the [`postgresql`](http://community.opscode.com/cookbooks/postgresql). It can install mysql or postgresql and should allow us later to use a master/slave configuration.
