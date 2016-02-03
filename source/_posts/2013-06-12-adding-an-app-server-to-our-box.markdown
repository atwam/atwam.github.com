---
layout: post
title: "From pow to a deployed rails app using chef, capistrano and vagrant - Part 3"
date: 2013-06-12 08:47
comments: true
categories: 
---

Now that we have a server running our database, we want to make it also able to run our rails app.
As I said in the previous post, I could use the full `application` cookbook, but I'd have to create my
own cookbook and recipe to use that properly. For now, I'll just rely on `rackbox` (and its dependency `appbox`).

## Managing with `appbox`

The `appbox` cookbook does pretty much the stuff that we configured earlier (setting up users etc). Since it's a dependency of `rackbox`,
we are pretty much forced to use it (`rackbox` does call `appbox` default recipe).

Having added `appbox` to our `Berksfile`, I had to modify the `roles/base.rb` :

* Let app box know that my admin user name is wam.
* Generate a ssh key for to login as the deploy user.
* Make sure the app box recipe is run before I run my `sudo` and `users::sysadmins` recipe.

We'll start by creating the deploy key :
```bash
$ ssh-keygen -t rsa -C "wam@scube" -f ~/.ssh/scube_deploy_rsa
```

```ruby roles/base.rb
name "base"
description "Base role applied to all nodes."
run_list(
  "recipe[apt]",
  "recipe[build-essential]",
  "recipe[git]",
  "appbox",
  "recipe[users::sysadmins]", # Necessary to run after appbox to add our stuff
  "recipe[sudo]" # Same
)
override_attributes(
  "appbox"  => {
    "deploy_keys" => [
      "ssh-rsa [...]" # content of ~/.ssh/scube_deploy_rsa.pub
    ],
    "admin_user" => "wam",
  }
)
```

## Adding a rack server
I've chosen to use passenger + nginx, which is a popular choice among the rails community. I was tempted for a moment by puma on jruby, but I want
my app online faster and will bother changing this kind of thing later (chef makes it easy to test new nodes with new recipes..)

Let's create a `roles/app_server.rb` :

```ruby roles/app_server.rb
name "app_server"
description "Serving http requests for the app. Main app server"
run_list(
  "rackbox"
)
override_attributes(
  "rackbox" => {
    "ruby" => {
      "versions"=> ["1.9.3-p385"],
      "global"=> "1.9.3-p385"
    },
    "apps"=> {
      "passenger"=> [
        {"appname"=> "scube", "hostname"=> "my.hostname.com"}
      ]
    }
  }
)
```

Now to make it work with vagrant, two changes are necessary in our `Vagrantfile` :

* We need to make sure that the `chef` version we are using is 11 or more. By default my vagrant was using chef 10, and the `rackbox`
(more specifically the `runit` it uses) was throwing an error (NameError: Cannot find a resource for load\_new\_resource\_state on ubuntu version 12.04).
* We add a port mapping to access the http port of our server on `localhost:8888`

```ruby Vagrantfile
# Stuff here
config.vm.network :forwarded_port, guest: 80, host: 8888
# Put this line just before your config.vm.provision :chef_solo line
config.vm.provision :shell, :inline => "gem install chef --version 11.4.2 --no-rdoc --no-ri --conservative"
```

The `:shell` provision makes sure that vagrant updates the chef gem before actually running our `chef-solo` provision.
Now it's on to `vagrant provision`, stuff should appear in green and [http://localhost:8888](http://localhost:8888) should show a 404 error.
Yes of course, we haven't deployed our app yet. That'll be next.
