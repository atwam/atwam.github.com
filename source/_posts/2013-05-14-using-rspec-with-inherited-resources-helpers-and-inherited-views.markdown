---
layout: post
title: "Using Rspec with inherited_resources helpers and inherited views"
date: 2013-05-14 19:51
comments: true
categories: 
---

I have updated a rails app I have been working on recently to a more recent version of rails 3.2, and all my tests where failing.
Finally managed to have that working, figured I'd show how.

### Mocking inherited_resources helpers in views specs.

I know I shouldn't be using [`inherited_resources`](https://github.com/josevalim/inherited_resources) anymore ([see here](http://stackoverflow.com/questions/9599201/inherited-resources-deperecated-on-rails-3-responders) and [here](http://archives.ryandaigle.com/articles/2009/8/10/what-s-new-in-edge-rails-default-restful-rendering)) but I want to release my app before I change everything to use responders.

So, my tests where failing because I was using the `resource`, `collection` and `resource_class` helpers from some views I was using. So first my tests are failing because `resource_class` isn't available in my views. I would have thought that the controller helpers were available in the views, but they aren't.
The solution is easy. Let's add the following to our `spec/support` directory :

```ruby spec/support/view_resource_macros.rb
module ViewResourceMacros
  def has_resource(name, &block)
    before do
      # Creates the resource
      @resource ||= yield
      # Assign to the symbol we wanted, so it's available in the view
      assign(name, @resource)
      # Assigns to @name so that we can use that in our assertions
      instance_variable_set("@#{name}", @resource)

      # If we pass an array, it's for stubing a collection, if not it's for stubbing a single object
      if @resource.is_a?(Array)
        view.stub(:collection) { @resource }
        view.stub(:resource_class) {@resource.first.class}
      else
        view.stub(:resource) {@resource}
        view.stub(:resource_class) {@resource.class}
      end
    end
  end
end

RSpec.configure do |config|
  config.extend ViewResourceMacros, :type => :view
end
```

And see how to transform our old (failing) test :
```ruby spec/views/cars/edit.html.haml_spec.rb
require 'spec_helper'

describe "cars/edit.html.haml" do
  before(:each) do
    assign(:car, @car = Factory.create(:car))
  end

  it "renders the edit view" do
    render
    rendered.should contain(@car.name)
  end
end
```
becomes :

```ruby spec/views/cars/edit.html.haml_spec.rb
require 'spec_helper'

describe "cars/edit.html.haml" do
  has_resource(:car) { Factory.create(:car) }

  it "renders the edit view" do
    render
    rendered.should contain(@car.name)
  end
end
```

### Using shared inherited partial in our views specs
Rails 3.1+ offers views inheritance, so I changed my code to have the following :

```ruby
class BaseController

end

class CarsController < BaseController
  inherit_resources
end

class PlanesController < BaseController
  inherit_resources
end
```

Then I just created a `base/new.html.haml` and a `base/edit.html.haml` views, to use the views inheritance.

```haml base/new.html.haml
%h1 Create #{resource_class.model_name.human}
= render :partial => "form"
```
```haml base/edit.html.haml
%h1 Edit #{resource}
= render :partial => "form"
```

And I have two `_form.html.haml` partials, one for each controller.
Now the next issue is that our edit and new views are shared, but we still want to test the `_form.html.haml` partial.

```ruby spec/views/cars/_form.html.haml_spec.rb
require 'spec_helper

describe "cars/_form.html.haml" do
  {
    new: -> { Car.new }
    edit: -> { Factory.create(:car) }
  }.each do |name, block|
    context "when called in ##{name}" do
      has_resource(name, block)
      
      it "renders the form" do
        render
        rendered.should have_selector("form")
      end
    end
  end
end
```

### Shared partial
Finally, when testing for example `cars/index.html.haml` which uses a partial `toolbar.html.haml` that actually exists in `base` views, the following lines are necessary :
```ruby
before do
  views.lookup_context.prefixes << "base"
end
```

This was [raised as an issue to the rails team](https://github.com/rails/rails/issues/5213), but they commented (rightly I think) that the inheritance
is related to the controller, not the views, so the test case shouldn't know about it and you'll have to declare it manually using the lines above.

Now let's go back and make these tests green again.
