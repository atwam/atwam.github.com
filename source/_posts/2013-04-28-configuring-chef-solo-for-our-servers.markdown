---
layout: post
title: "From pow to a deployed rails app using chef, capistrano and vagrant - Part 2"
date: 2013-04-27 23:03
comments: true
published: false
categories: 
---

Now that we have our vagrant box working with chef, let's use chef to configure our services and our app.

## Chef roles

Assuming that you have read some basics about chef, you'll know that the cookbooks we have downloaded provide recipes for installing various software.
We could ask vagrant to install a few recipes, but it's probably better to assemble them in roles.
We'll then assign these roles to one or several nodes, or use all of them on our box for testing.

For now, we probably want to have one `base` role (to install common software on all our nodes) and two roles to serve our application :

database\_master
: a simple install of postgres should be fine here.

app\_server
: this one will serve our RoR app.

One could think of other roles (workers, redis etc), but for my purpose and for now these two (and the `base` role) should be fine.

## Base role

We want our base role to include the following :

* *apt*, *git*, *sudo* and *build-essential* should be installed. We'll use the default cookbooks/recipes for each of these.
* *users setup* : Should create the users (with their ssh key), give them sudo rights. We'll use the `users` cookbook.

We start by editing our `Berksfile` to make sure all the cookbooks are included (`sudo`, `apt`, `git`, `build-essential`, `users`).
Then let's create a role file.

```ruby roles/base.rb
name "base"
description "Base role applied to all nodes."
run_list(
  "recipe[apt]",
  "recipe[build-essential]",
  "recipe[git]",
  "recipe[users::sysadmins]",
  "recipe[sudo]"
)
```

*The order matters here !* _apt_ should appear first (it's used to handle packages), _build-essential_ is used by pretty much
everything, and especially by `ruby-shadow` which is a gem dependency of `users`.

Reading [the documentation of the users cookbook](http://community.opscode.com/cookbooks/users), we see that we should define the users in a data bag
(a way of telling chef about some data, list, including potentially encrypted password and ssh keys).

Chef solo doesn't work very well with data bags (or the CLI doesn't work very well), so we'll just create the file manually.
Also, we see in the users cookbook that it requires `chef-solo-search` to run with chef-solo.

Adding `cookbook 'chef-solo-search', git: "https://github.com/edelight/chef-solo-search.git"` to our `Berksfile` should be good enough.

```bash
$ mkdir data_bags/users
$ vim data_bags/users/wam.json
```
```json data_bags/users/wam.json
{
  "id": "wam", //or your user name
  // The following should be the result of openssl passwd -1 plainpasswd
  // but that's md5 on a mac. Alternatively run mkpasswd -m sha-512 -S mySalt on a linux machine
  "password": "$6$[...]098/",
  "ssh_keys": [
    // Copy paste from your ssh public key
    "ssh-rsa AAA123...xyz== foo"
    ],
  "groups": [ "sysadmin" ],
  "uid": 2001,
  "shell": "\/bin\/bash",
  "comment": ""
}

```

Now we need to modify our `Vagrantfile` to use this role (and not the dummy git recipe we were using). An extra bit of precaution is needed here :
the `sudo` cookbook/recipe will install `sudo` qnd configure it by default for the sysadmin group (lucky us, our user is a member).
*It will override vagrant's sudo config, breaking vagrant provision using chef-solo*. To avoid that, we use vagrant's `chef.json` config
to override the `sudo` configuration attributes for vagrant :

```ruby
  chef.add_role "base"
  chef.json = {
    :authorization => {
      :sudo => {
        :users => ["vagrant"],
        :passwordless => true
      }
    }
  }
```

Then it's on to `vagrant provision`, and ssh to whatever port was forwarded to 22 (for me it was `ssh localhost -p 2222`) to see that you log in
using your ssh key.

If you hit a json parsing exception when chef reads your user json file, make sure you don't have trailing commas.
You can check your JSON easily in `irb` using `require 'json'; JSON.parse(File.read('data_bags/users/wam.json'))`.

## Creating a custom cookbook

Most of the cookbooks we use provide LWRPs, that is a way to simply create what we need using a ruby script, which allows a greater flexibility
than using simply our cookbooks default recipes.
It makes sense to create a cookbook for our app, where we'll include any app specific configuration/templates etc.



## Database master role

Now that we have the base configuration for our servers (we'll probably modify it in a bit, once we have created our own custom cookbook),
we need to create our `database_master` role. We'll make a heavy use of the [database cookbook](http://community.opscode.com/cookbooks/database)
so make sure you have read its documentation.




### References
[Rails quick start](http://wiki.opscode.com/display/chef/Build+A+Rails+Stack) ([github](https://github.com/opscode/rails-quick-start))
