---
layout: post
title: "From pow to a deployed rails app using chef, capistrano and vagrant - Part 2"
date: 2013-06-10 23:03
comments: true
categories: rails, chef, capistrano
---

Now that we have our vagrant box working with chef, let's use chef to configure our services and our app.

## Chef roles

Assuming that you have read some basics about chef, you'll know that the cookbooks we have downloaded provide recipes for installing various software.
We could ask vagrant to install a few recipes, but it's probably better to assemble them in roles.
We'll then assign the roles to one or several nodes, or use all of them on our box for testing.

For now, we probably want to have one `base` role (to install common software on all our nodes) and two roles to serve our application :

* `database\_master` : a simple install of postgres should be fine here.
* `app\_server` : this one will serve our RoR app.

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

### Creating a custom cookbook ...
 
There's a big choice to do here. We could either create a whole separate cookbook just for our app, configured with many
default recipes, or for now just use an already created one.

It is very likely that I'll have to create a cookbook at some point, because it's the only way to have your own recipes
and reach a high enough level of customization.

### .. or use and existing one
I originally had a look at the [database cookbook](http://community.opscode.com/cookbooks/database) but finally decided
to go the fast way by using two very neat cookbooks, [rackbox](https://github.com/teohm/rackbox-cookbook) and [databox](https://github.com/teohm/databox-cookbook).
It will probably make sense to use `database` and `application` cookbooks, but they seem to be easier to work with when you are using a proper chef server
and your own cookbook/recipes.

`rackbox` includes `appbox` by default, which creates its own users for deployment/app running.
I have found that these cookbooks are a bit limited for my taste (for example, they don't use data\_bags, which are a proper way of encrypting
password instead of storing them in your chef repository... Well, next time.

## Setting up our roles

Let's start by adding the cookbooks to our `Berksfile` and run `berks install`
```ruby Berksfile
cookbook "runit", ">= 1.1.2"  # HACK: force-use this version
cookbook "databox"
cookbook "rackbox"
```

and create our `roles/database_master.rb`. We are using non encrypted passwords here, which isn't very secure.
We should actually use encrypted data bags, but they don't play very nicely with roles (they are supposed to be used with recipes, which
would mean custom cookbook), nor do they play nicely with `knife solo` (although a plugin exist, but it didn't work very well
in my tests). Let's start this way, we'll see later to move to a more robust non solo chef.

```ruby roles/database_master.rb
name "database_master"
description "Master postgresql node"

run_list(
  "databox::postgresql" # Or "databox" to include mysql as well
)
default_attributes(
  :databox => {
    :db_root_password => "PASSWORD_HERE",
    :postgresql => [
      {
        "database_name" => "myapp_production",
        "username" => "myapp",
        "password" => "ANOTHER_PASSWORD_HERE"
      }
     ]
  }
)
```

Now running `vagrant provision` (or `vagrant up` or `vagrant reload` depending on whether your current vagrant box is up or not) should run this recipe, adding
the `myapp` database. We can test that in `vagrant ssh`
```bash
$ psql -h localhost -d myapp_production -U myapp -W
Password for user myapp:
#psql (9.1.9)
#[...] Yeepee
```

## What's next

Next post will be about configuring a proper rails box using `rackbox`, setting up capistrano to deploy ... then deploy to a vps and get closer to production.
I'm still not entirely happy with this deployment today. I should move to a proper cookbook, as I said, to get more customization options.
For now, I want my app out, and will probably work a bit more later depending on how successful it is. The beauty of chef, after all, is that it makes
it easy to set up new nodes and new deployments.

### References
[Chef cookbooks for busy ruby developers](http://teohm.github.io/blog/2013/04/17/chef-cookbooks-for-busy-ruby-developers/)

