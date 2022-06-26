---
layout: post
title:  "Ruby constant lookup: compact style vs. explicit nesting"
date:   2022-06-25 12:00:00 +0200
categories: coding
---

Recently, a colleague of mine decided that it was time to refactor some parts of our (Rails-) codebase. In particular, he used the [autocorrect feature of rubocop](https://docs.rubocop.org/rubocop/usage/auto_correct.html) to switch from qualified constant definitions (aka. compact style) to explicit nesting:

{% highlight ruby %}
# Before
class Product::Operation::Create
  # ...
end

# After
module Product
  module Operation
    class Create
      # ...
    end
  end
end
{% endhighlight %}

Unbeknownst to me, this seems to be a [ruby style guide recommendation](https://github.com/rubocop/ruby-style-guide#namespace-definition) because using compact style can lead to "*surprising constant lookups*". This sounded good in my ear, so I approved his pull request.

**But there was one commit that was bothering me.** The commit message said:
```
Fix regressions resulting from enforcing nesting style

* Fix resolving `Contract` by prefixing `self::`.
```

Prepending `self::`? Never had I ever seen that `self` was being used to specify a qualified constant. I was clueless to why this worked and albeit having been programming in ruby for 5+ years, I never reached the level of profound language understanding that I used to have in JavaScript. Rails abstracts most internals away, so... I never really cared. But now, I felt, was the time to dig in. Before I'm going to present you my findings, let me provide you with some...

## Context

The Rails app in question is a version 6.1 monolithic monster, which makes heavy use of the [trailblazer](https://trailblazer.to/) and [dry-validation](https://dry-rb.org/gems/dry-validation/) libraries to abstract away business logic and slim down models and controllers. The paradigm applied here is called DCI (domain - context - interaction). If you're not familiar with it, I recommend reading [Aref Aslani's article](https://medium.com/swlh/better-rails-service-objects-with-dry-rb-702687394e3d), since explaining it is beyond the scope of this post.

Before the refactor, an `Operation` and a corresponding `Contract` would look like this:

{% highlight ruby %}
# Example Operation
class Product::Operation::Create < Trailblazer::Operation
  extend Contract::DSL  

  step     Model( Product, :new )
  step     Contract::Build()
  step     Contract::Validate()
  failure  :log_error!
  step     Contract::Persist()

  # ...
end

# Example Contract
module Product::Contract do
  Create = Dry::Validation.Form(Validation::BaseSchema) do
    required(:name).filled(:str?)
    required(:price).filled(:int?)
    # ...
  end
end
{% endhighlight %}

After the switch to **explicit nesting**, the classes looks like this:

{% highlight ruby %}
class Product
  module Operation
    class Create < Trailblazer::Validaton
      extend self::Contract::DSL  

      step     Model( Product, :new )
      step     self::Contract::Build()
      # ...
    end
  end
end

class Product
  module Contract
    Create = Dry::Validation.Form(Validation::BaseSchema) do
      # ...
    end
  end
end
{% endhighlight %}

As you can see, my co-worker prepended `self::` to `extend Contract::DSL` and all `step` calls invoking functions found in `Contract`. When asked why this was needed, he responded: "Without `self::` the error `uninitialized constant Product::Contract::DSL` get's thrown. And added, that honestly speaking, he doesn't know what's up.

I was puzzled. From what I knew about constant lookup in Ruby, prepending `self::` should do nothing, as `self` is referring to currently open class or module - and that is always the starting point from where the interpreter tries to discover the constant in question.


## The possible difference

Before finding out how this unusual `self::` fix functioned, I needed to grasp why this code broke after all. After reading the infamous [constant lookup blog post](https://cirw.in/blog/constant-lookup.html) by Conrad Irwin, I found out that the difference between explicit nesting and compact style manifested itself in the contents of `Module.nesting`. This method returns an array which contains a stack of objects representing the actual nesting. In other words, this array contains the [lexical scope](https://stackoverflow.com/questions/1047454/what-is-lexical-scope) of the current execution context.
Apart from the lexical scope, Ruby uses the ancestors of the currently open module or class to find a constant. But we'll come to that later, for now let's continue examining `Module.nesting`.

Explicit nesting pushes multiple objects to `Module.nesting` because:

* The class object following a class keyword gets pushed when its body is executed, and popped after it
* The module object following a module keyword gets pushed when its body is executed, and popped after it.

*NOTE: There are some other ways to push objects to Module.nesting in Ruby, the [Rails autoloading guide](https://guides.rubyonrails.org/v6.1/autoloading_and_reloading_constants_classic_mode.html#nesting) provides a solid overview.*

That means that, when using explicit nesting for our operation, `Module.nesting` will contain:
{% highlight ruby %}
[Product::Operation::Create, Product::Operation, Product]
{% endhighlight %}

Compact style only pushes one object to `Module.nesting`, because there is only one class keyword here. `Module.nesting` will only contain:
{% highlight ruby %}
[Product::Operation::Create]
{% endhighlight %}

## Constant lookup order
*Now, if that is the only difference, I suspect that this is where something goes wrong*, I said to myself. Looking into the `trailblazer-operation` gem (V2.0, because outdated legacy code), I discovered that `Contract` was a module in `Trailblazer::Operation`, which has a submodule called `DSL`. This is what we wanted! Our Create Operation inherits from `Trailblazer::Operation`, so why isn't `Contract::DSL` fetched from the operation's ancestor?


### Lexical scoping first
The reason is that lexical scoping kicks in before Ruby looks into the ancestor chain of our currently open class / module. After reading this fact, I suddenly had a realization: We have a `Contract` module in `Product`! That means that `Contract` is resolved to `Product::Contract` because of lexical scoping.
This error had not been there before, because `Product` had not been in `Module.nesting` when we were still using compact style.

### Forcing our way into the ancestor chain

How can we "skip" lexical scoping to make sure we load the right `Contract` module? I found the solution, but not in Irwin's article, but in the [Rails autoloading guide](https://guides.rubyonrails.org/v6.1/autoloading_and_reloading_constants_classic_mode.html#resolution-algorithm-for-qualified-constants):

*1. The constant is looked up in the parent and its ancestors. In Ruby >= 2.5, `Object` is skipped if present among the ancestors. `Kernel` and `BasicObject` are still checked though.*

*2. If the lookup fails, `const_missing` is invoked in the parent. The default implementation of const_missing raises `NameError`, but it can be overridden.*

First of all, this explains the error that we got. When Ruby tried to look up `Contract::DSL`, it searched for `DSL` in `Contract` and its parents. `Contract` didn't contain `DSL`, nor did its parents, so `const_missing` was invoked in it.

Now what happens when we write `self::Contract::DSL`? `Contract` will become a qualified constant by itself. That means, that it's only looked up in the parent and its ancestors - which is our `Product::Operation::Create` and it's ancestor `Trailblazer::Operation`! **To sum up: We successfully skipped lexical scoping by prepending** `self::`.


# Wrap up

To cite Conrad Irwin, *constant lookup in Ruby isn’t actually that hard after-all*. Once you get a hang of it, you can easily deduce why lookup errors happen or autoloading fails. For me, this case was especially confusing because of the `self::` syntax - I think a clearer solution would have been to just explicitly state `extend Trailblazer::Operation::Contract` to avoid any misconceptions.

I highly recommend you read [Irwin's article](https://cirw.in/blog/constant-lookup.html). Especially the code snippet in the summary, where he recreates the constant lookup algorithm in ruby itself, was truly helping me understand the order in which constants are resolved.

