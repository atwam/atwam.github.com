---
layout: post
title: "From pow to a deployed rails app using chef, capistrano and vagrant - Part 1"
date: 2013-04-27 22:09
comments: true
categories: [rails, chef, vagrant, capistrano]
---

So, let's be honest, I've been quite lazy here. My latest rails application was deployed with Heroku. Lots of fun, a very pleasant experience.
But now I have this new shinny application that I don't want to deploy using Heroku :

* Because I am worried that if I start with Heroku, I'll be too lazy to switch later.
* Because I know that at some point, my application is going to need plugins and binaries that I can't get on Heroku.
* Because I'd rather have a portable application that I can deploy easily on any type of server.

<!-- more -->

## 1. Planning and strategy
I have this nice application, let's call it cube. So far it has lived very happily in rails dev environment, run only by the marvelous [pow](http://pow.cx).

My goal is to have :

* A server configuration using chef
* An easy deployment using capistrano
* A way to locally run a copy of the server deployment using vagrant

This should allow me to easily change/add services on my box, and scale it up should the need arise.
So, the plan is the following :

1. Install `vagrant`, that I'll use to launch virtualized servers to test my chef configuration.
2. Prepare a chef configuration for two roles : app server and database server. 
I'll start by having the same machine (in vagrant for now) running the two roles, knowing that I'll be able to split that to more servers if I need to scale.
3. Setup capistrano on my rails app to have it deploy nicely to my machines.

## 2. Vagrant for testing locally the server configuration

Well, it'll be short. Head over to [vagrant's website](http://www.vagrantup.com) where its marvelous developper has done a very complete getting-started guide.
Once installed (and don't forget to install virtualbox too), I'm pretty much following it.
I'll chose the `precise64` box, an ubuntu 64 bits image, but you can of course choose another box found [here](http://www.vagrantbox.es/)

I'll start by creating a directory that will hold our deployment related configuration : vagrant and chef's config.
I have chosen to have this directory separate from my application's directory.

```bash
$ mkdir deployment && cd deployment
$ vagrant init
$ vagrant box add precise64 http://files.vagrantup.com/precise64.box # Downloads the box
```

Then let's edit our `Vagrantfile` to change a few defaults to better suit us :

```ruby Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"
  # This line ensures that we'll know where to find the box should we clone our deployment repo on another computer
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"
end
```

We just start the box with `vagrant up` and check that we manage to connect to it using `vagrant ssh`. It's working, great, we have a blank slate to work on !

## 3. Chef in the kitchen

[Chef](http://www.opscode.com/chef/) will be used to handle the configuration/bootstrapping of my servers.
I've done my fair share of playing with ssh in the past, only to realize that administering a server for a side project using ssh is fun until
you have to come back to it after a few months and remember where all the things are and where you should do you change.

With chef, the configuration is stored on git, and if I need to change anything, there's only one place to go.
Soooo, let's start.

```bash
$ # I'm still in my deployment directory that I created earlier
$ cat > Gemfile <<EOF
source :rubygems

gem "chef" # The main gem
gem "knife-solo" # Chef solo to bootstrap/configure chef on individual machines through ssh
gem "berkshelf" # To manage cookbooks
EOF
$ bundle install
```

If you see an error requesting gecode when you run `bundle install` then `brew install --use-llvm gecode` should be good enough to make it work.
It may be a bit more complicated than that, just look at [this blog post]({% post_url 2013-04-25-building-dep-selector-on-mac-os-x %}) if you have issues installing the gems above.

That should cover the installation of the main `chef` gem and of `knife-solo`. Chef can be configured to use with a chef server (useful to manage many machines)
but for now I'll be using `knife-solo` (also known as chef-solo) which should be good enough.

Now let's initiate a chef repository in my directory. This repository will hold all the configuration for the server(s) I am going to deploy to.

```bash
$ knife solo init . # Creates the chef-repo here
```

If you have read a bit about chef (of course you have), you know that one of the main ingredients (ahah) are the *cookbooks*.
Opscode has gathered many cookbooks contributed by the community.
The standard way of installing these cookbook is through `knife cookbook site install blah`, but I have included the [berkshelf](http://berkshelf.com/) gem which will handle
our `cookbooks` directory for us and get all the proper cookbooks and their dependencies. The list of cookbooks is managed in a `Berksfile`.

For now, let's start by installing some cookbooks. We'll see later to configure them and our app.

```ruby Berksfile
site :opscode

cookbook 'apt'
cookbook 'build-essential'
cookbook 'nginx'
cookbook 'database'
cookbook 'git'
```
A few comments on these cookbooks ..

apt
: Used to make sure this package manager is installed on the ubuntu box that we'll be using.

build-essential
: Make sure we can build stuff. This one will be included anyway since it's a dependency of for many other packages.

[nginx](http://community.opscode.com/cookbooks/nginx)
: Planning to use this lightweight web server to serve our rails app

[database](http://community.opscode.com/cookbooks/database)
: This cookbook is an alternative to the [`postgresql`](http://community.opscode.com/cookbooks/postgresql). It can install mysql or postgresql and should allow us later to use a master/slave configuration.

Now let's make sure these cookbooks are there and available :
```bash
$ berks install
```

## 4. Make vagrant use chef

Now we'll need to make vagrant use our chef cookbooks and recives, so that we'll be able to test our chef configuration.
For now we'll just make sure it installs correctly a simple recipe (git, for example). We'll cover the proper configuration
of our servers in a later post.

Since we are using berkshelf to manage our cookbooks, we'd better use the `vagrant-berkshelf` plugin :
```bash
$ vagrant plugin install vagrant-berkshelf
```

Now let's add a few lines in our `Vagrantfile` :

```ruby Vagrantfile
Vagrant.configure("2") do |config|
  # [snip] our previous config
  #
  config.berkshelf.enabled = true
  config.vm.provision :chef_solo do |chef|
    # The following is not necessary because we use berkshelf
    # chef.cookbooks_path = "cookbooks"
    chef.roles_path = "roles"
    chef.data_bags_path = "data_bags"
    
    # For now let's just install git to check that our chef provisioning is working
    chef.add_recipe "git"

    # We'll use the following later
    # chef.add_role "web" 
  end
end
```

Now we need to `vagrant reload` to make vagrant recreate a machine and install the required chef folders,
then `vagrant provision` to make chef-solo install whatever is needed. A `vagrant ssh` followed by a `git --version`
should help you verify that our chef provisionning worked.

You can see in our `Vagrantfile` above that we just asked chef-solo to run the `git` recipe.
Looking at [git cookbook's documentation](http://community.opscode.com/cookbooks/git),
we can see that this will run the `git::default` recipe of this cookbook, which should just install git.

Let's remember that in the future running `vagrant provision` will rerun chef on our vm, so we'll
be able to quickly test our changes in config.

## 5. Server and app configuration with chef
I'll cover that in my next post.

### References
[Practical chef and capistrano for your rails app](http://www.slideshare.net/SmartLogic/practical-chef-and-capistrano-for-your-rails-app)
