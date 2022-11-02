# Oxidizer

Rails controllers require a lot of boilerplate for non-trivial Rails applications. Oxidizer reduces a lot of the boilerplate and spaghetti code typically seen in Rails controllers by:

1. Moving authorization out of controller methods and callbacks and into policy objects via Pundit.
2. Encourage the use of more, but smaller, controllers to handle various interactions with ActiveRecord objects and other Resources.
3. Utilizes PORO and inheritance for making controller code less verbose, as opposed to a DSL approach, which can be difficult to extend and obfuscates how Rails controllers work.
4. Encourages keeping business logic out of ActiveRecord objects **and** controllers by utilizing Resource objects.

Putting that all together, a typical Oxidizer controller that handles CRUD actions for a blog comment feature would look like this:

```ruby
# Example Oxidizer controller for comments in a blog post.
class CommentsController < Oxidizer::NestedResourcesController
  protected
    def self.resource
      Comment
    end

    def self.parent_resource
      Post
    end

    def assign_attributes
      @comment.user = current_user
      @comment.blog = @post
    end

    def permitted_params
      [:post_id, :body]
    end
end
```

Since there's no DSLs, its easy to extend Oxidizer controllers to implement any type of behavior you need.

## Installation

Add to your Rails application Gemfile by executing:

```bash
bundle add "oxidizer"
```

Then run:

```bash
# TODO: Not implemented yet
rails generate oxidizer:install
```

This will create the folders and files needed to get going with Oxidizer.

```txt
# TODO: Not implemented yet
app/controllers/application_resources_controller.rb
```

The application resources controller is actually a few controllers. You can split them out, but to start its easiest to keep them in one file.

```ruby
class Resources < ApplicationController
  include Oxidizer::ResourceCollection

  class Resource < Resources
    include Oxidizer::SingularResource
  end

  class NestedResource < Resource
    include Oxidizer::NestedResource
  end

  class NestedWeakResource < Resource
    include Oxidizer::NestedWeakResource
  end
end
```

Then from your application, you can generate resources as follows:

```sh
./bin/rails g oxidizer:resources
```

## Routing

Oxidizer can automatically mount your resource tree so you don't have to maintain redudant route files. Here's how that looks:

### Some modest routing extensions

Given a Rails routing hierarchy like this:

```ruby
resources :items do
  get :search, to: "items/searches#index"
  scope module: :items do
    resources :children, only: %i[index new create] do
      collection do
        get :templates
      end
    end
    resources :ancestors, only: %i[index]
    resources :labels, only: %i[create]
    resources :copies, only: %i[create new]
    resources :batches, only: %i[new create]
    resource :icon, only: %i[edit update]
    resource :movement, only: %i[new create]
    resource :loanable, only: %i[new create], controller: "loanable_items"
    template_resources :containers, :items, :perishables
  end
end
```

It might be clearer with something like this:

```ruby
resources :items do
  get :search, to: "items/searches#index"
  dir :items do
    resources :children, only: %i[index new create] do
      collection do
        get :templates
      end
    end
    list :ancestors
    edit :icon
    create :labels
    create :copies
    create :batches
    create :movement
    create :loanable, controller: "loanable_items"
  end
end
```

The names more clearly show the relationship of the controller to its resource, what what it's expected to do.

### Routing trees

```ruby
oxidizer.resource :accounts
```

If the structure of accounts is:

```
accounts
  |- items_controller.rb
  |- labels_controller.rb
accounts_controller.rb
items_controller.rb
```

All routes will get automatically generated. There's no need to maintain a Routes file that mimics the structure of your controllers.

### Steal Bullet Train routes

I'm probably going to steal [Bullet Train Routes](https://github.com/bullet-train-co/bullet_train-routes), which will look something like this:

```ruby
model "Orders::Fulfillment" do
  model "Shipping::Package"
end
```

But instead I'd pass controllers into it, like this:

```ruby
# Heh, can't think of a name ATM and `controller` is taken.
kontroller Orders::FulfillmentController do
  kontroller Shipping::PackageController
end
```

It's tempting to take this a step further and delegate routing to a folder full of controllers, that might look something like this:

```ruby
controller_hierarchy Account
# Iterates through Account::Users, Account::Items, Account::Blah... and connects all of the resource routes.
```

Regardless, any shortcuts taken should interoperate with Rails controllers in case the developer needs "eject" from the abstraction and configure "vanilla" Rails routing. This is the key thing that Bullet Train Rails gets right.

#### CRUD routes

Another option that would be fun for routes, which I'm not leaning towards at the moment, are `crudi` routes.

```ruby
crud Orders::FulfillmentController do # Create, read, update, & destroy
  cru Shipping::PackageController # Create, read, & update: no destroy
end
```

This is probably "too clever"; a novice won't know what CRUD means. This would be a fun way to deal with the `resources :blah, only: [:create, :new, :update, :edit]` business.



## Concepts

Oxidizer makes it easy to build RESTful Rails applications that follow the CRUD controller pattern and shallow routes.

## Controller types

There's a few types of controllers you'll want to use:

### ResourcesController

The most common type of controller is a resources controller. Its very much like a vanilla RESTful Rails controller where `index` is the collection of resources and `new`, `create`, `show`, `edit`, `update`, and `destroy` operate on the singular resource.

For example, a blog web application might have a `Posts` Resources controller.

### ResourceController

Similar to above, but does not have an `index` action. Singular resources are commonly used in web applications for managing the current users profile and associated resources.

For example, a blog web application might have a `Session` Resource controller that the user can create when they login and destroy when they log out.

### NestedResources

Nested resources are designed to be scoped within a `Resources`. They have `new`, `create`, and `index` actions, but do not have the remaining actions. The remaining CRUD actions for a nested resource should be `Resources` controller.

For example, a blog's `Post` resources might have many `Comment` resources per post. The creation of the comment is within the context of the `Post` resource. After the `Comment` resource is created, the `Post` should be persisted in the `Comment` (probably as `comments.post_id`) if it needs to be accessible after its persisted.

It's possible to have the other CRUD actions in a nested resource, but its discourage since nesting controller scopes can be difficult to maintain as dependencies and business logic change. Best to keep themn flat.

### NestedResource

A nested resources is similar to the nested resources, but is singular. For example, a `Post` may have an `Author` resource at `posts/:id/author`. The singular nested resource supports the full range of CRUD actions, but does not have `index`.

### NestedWeakResource

A nested weak resource is on where the underlying resource is the same as the parent resource.

For example, a `Post` may require a confirmation screen before its deleted available at `posts/:id/delete_confirmation/new`. The user would press the `Confirm deletion` button on that screen which would `POST` to `/posts/:id/delete_confirmation` and destroy the object.

## Contributing

Open issues with reproducible steps.

## Compared to other gems

During the development of this gem, some of these other libraries were mentioned as "similar gems", so I evaluate each one to make sure I'm not implementing the same thing or communicate "how its different this time".

### [Inherited Resources](https://github.com/activeadmin/inherited_resources)

> Sounds a bit like inherited_resources tbh?
>
> I get the feeling I’ve already been there and didn’t like it in the end.

[@julian_rubisch](https://twitter.com/julian_rubisch/status/1587186996879007747?s=61&t=gOFfW6GtJqGs5ETPy_nsog)

Yeah, I've been there too! I used the Inherited Resources gem once for an admin panel and found it difficult to do things when I inherited.

Take for example this:

```ruby
# Code from InheritedResources
class AccountsController < InheritedResources::Base
  defaults :resource_class => User, :collection_name => 'users', :instance_name => 'user'
end
```

That's not really inheritence; rather, it's a class method DSL that configures instances methods. Why not just modify the instance methods themselves?

That's how Oxidizer does it:

```ruby
class AccountsController < ApplicationResourcesController
  def self.resource
    User
  end

  def resource_name
    "user"
  end

  def resources_name
    "users"
  end
end
```

It's inheritence how you'd expect it to work; not a DSL.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
