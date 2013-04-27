---
layout: post
title: "Building dep_selector on mac os x"
date: 2013-04-25 23:13
comments: true
categories: [ruby, gem, chef]
---

Just a quick post, having finally figured out how to install dep\_selector on mac os x.
The issue is that having xcode installed which configures `clang` as the default compiler, some native gems break.

It took me some time, but at long least I can now build a native gem using `gcc` instead of `clang` on mac os x.
And you'll see it's not very easy to have ruby change its compiler for gems native extensions compilation.
<!-- more -->

## Installing gecode

`brew install gecode` is supposed to be fine, but it failed for me. The solution was to use llvm :

```bash
$ brew install gecode --use-llvm
```

## Install dep\_selector

`dep_selector` uses the standard `mkmf` to create the makefile that will be used to build the native extension for the gem.
`mkmf` in turn uses `RbConfig`, which is defined in `rbconfig.rb` deep in your ruby source and was generated when your ruby was built.
The issue is that this `rbconfig` hard codes the value of the `CC` variable that will be used in the `Makefile`.

`gem install dep_selector` fails with compilation errors by clang.

* First solution : modify `rbconfig.rb` to use `ENV['CC']` if defined. Yeah, that'd be ugly.
* Second solution : manually build the extension and let rubygems know that our gem is installed.

```bash
$ cd $GEM_HOME/gems/dep_selector-0.0.8/ext/dep_gecode
$ # First let's confirm that this was built with the wrong compiler
$ grep clang Makefile
CC = clang
```

Now let's edit the `extconf.rb`, adding the following line just before the last (`create_makefile()` thing).
```ruby
%w{CC CXX}.each do |c|
  RbConfig::MAKEFILE_CONFIG[c] = ENV[c] if ENV[c]
end
```

And let's recreate the `Makefile` and build our extension.
```bash
$ CC=gcc CXX=g++ ruby extconf.rb
$ grep CC Makefile # Should output gcc, great success !
$ make
$ make install # This will copy the library so to a proper ruby directory
```

We can check that the compilation worked, because there is now a `dep_gecode.bundle` file.
Now we need to let the gem system know that our gem is added. 
[The manual for `gem install`](http://docs.rubygems.org/read/chapter/10#page33) tells us that we'll have to copy the gemspec. The following should do the trick :

```bash
$ cd $GEM_HOME/gems/dep_selector-0.0.8
$ gem spec --local ../../cache/dep_selector-0.0.8.gem --ruby > ../../specifications/dep_selector-0.0.8.gemspec
$ gem list dep_selector
*** LOCAL GEMS ***

dep_selector (0.0.8)
$ irb
>> require 'dep_selector'
=> true
>> exit
```

That's it, now we can move on and work on our chef install.
