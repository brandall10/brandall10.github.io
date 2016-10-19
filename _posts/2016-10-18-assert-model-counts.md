---
layout: post
title:  "Testing: Assert All Model Deltas in One Call"
date:   2016-10-18
categories: rails testing
permalink: /assert-model-counts
---

I have a love/hate relationship with ActiveSupport's [assert_difference](http://api.rubyonrails.org/classes/ActiveSupport/Testing/Assertions.html) as a tool to help validate db state.  On the one hand, this function is great for acceptance/integration tests where multiple tables may be affected.  On the other hand, its verbosity makes it rather untenable for even moderately complex transactions.  

Consider this:

```ruby
def some_service_test
  assert_difference ['User.count', 'Article.count'] do
    assert_difference 'Widget.count', 2 do
      assert_no_difference ['Role.count', 'Action.count'] do
        MyService.new(@service_params).call
      end
    end
  end
end
```

When MyService is called, this asserts that there is one new User and Article record each, 2 new Widget records, and no new Role or Action records.  This is actually a pretty benign case.  Is there a way we could make this easier to use?  

First off, imagine how useful it could be to pass around an assertion spec as a hash.  From the above we could pull out:

```ruby
@model_assertion_spec = { 
  'User.count' => 1,
  'Article.count' => 1,
  'Widget.count' => 2,
  'Role.count' => 0,
  'Action.count' => 0
}
```

Breaking out the assertion code in some_service_test, the code based on this spec could be written out as:

```ruby
def some_service_test
  assert_difference 'User.count', 1 do
    assert_difference 'Article.count', 1 do
      assert_difference 'Widget.count', 2 do
        assert_difference 'Role.count', 0 do
          assert_difference 'Action.count', 0 do
            MyService.new(@service_params).call
          end
        end
      end
    end
  end
end
```

A couple things to note

* We're only using one form of assert_difference, just one model and one count per block. 
* assert_no_difference is just syntactic sugar for assert_difference with a count of 0

This is certainly more verbose and probably not the thing we want to do as a rule.  But with a little metaprogramming magic, we can exploit this pattern to great benefit.

```ruby
def assert_hash_spec(hash_spec, &block)
  if current_spec = hash_spec.shift
    assert_difference(*current_spec) do
      assert_hash_spec(hash_spec, &block)
    end
  else
    yield
  end
end
```

We just shift off the first key/value pair of each hash entry and create an enclosing block. When we have no more entries, we invoke the code which will be in the innermost block. 

With this, our assertion chain is condensed down into a simple visual block:

```ruby
assert_hash_spec @model_assertion_spec do
  ServiceCall.new(@service_params).call
end
```

This is great, but our @model_assertion_spec is still a bit verbose.  This is because assert_difference is not designed for models per se, but to evaluate expressions passed in. How much nicer is it to have a model spec like:

```ruby
@model_assertion_spec = { 
  User: 1,
  Article: 1,
  Widget: 2,
  Role: 0,
  Action: 0
}
```

It's even more reasonable to break this out into a call if you wish.  Imagine:

```ruby
assert_model_diff User: 1, Article: 1, Widget: 2, Role: 0, Action: 0 do 
  MyService.new(@service_params).call
end
```

That 1% of the time you don't evaluate model deltas go ahead and use assert_hash_spec directly. The rest of the time, let's implement and use assert_model_diff:

```ruby 
def assert_model_diff(model_spec_hash, &block)
  hash_spec = Hash[ model_spec_hash.map{ |k, v| ["#{k.to_s}.count",v] } ]

  assert_hash_spec(hash_spec, &block)
end
```

And that's it. Hopefully this cuts out much cruft from your test suite and/or makes a more attractive case for validating model counts.
