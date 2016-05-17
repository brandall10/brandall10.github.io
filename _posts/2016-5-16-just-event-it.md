---
layout: post
title:  "Just Event It!"
date:   2016-05-16
categories: rails
permalink: /just-event-it
---

# ActiveRecord-like Hooks Anywhere? 

Lifecycle hooks (before_validate, after_commit, etc) are one of the most useful features of ActiveRecord.

What if I told you - with the power of Ruby meta-programming - that we can add such a facility to any Ruby object?

Let's start with a PORO (plain 'ol Ruby object) simply called MyPoro.  MyPoro is pretty basic, it just contains a method which prints out the sum of a range:

```ruby
class MyPoro
  def calculate_sum
    (lower..upper).to_a.reduce(:+)
  end
end
```

Now what if we wanted to have a nice message printed to the console, replete with an error if the sum exceeds a certain amount?


```ruby
class MyPoro
  MAX_SUM = 100
  
  def calculate_sum
    sum = (lower..upper).to_a.reduce(:+)
    
    puts "Your sum is: #{sum}"
    
    if sum > MAX_SUM
      $stderr.puts "Error: sum can no be greater than #{MAX_SUM}"
    end
    
    sum
  end
end
```

This is fine... except that calculate_sum is doing more than simply calculating a sum -- it's now concerned with formatting output and determining an error condition. Generally speaking this is work that would be better handled by the caller, but in this case we could argue this is logging.

Can we do better?  Let's think of calculating the sum as some event we want to track, and the result of tracking this activity is logging the details around it.  There are other things we may want to do as well - perhaps based on the results we could delegate some work to another object, update a database table, etc, just like one might do with ActiveRecord.  The sky's the limit.

# Creating an Event

So how do we do this?  Let's create a sum event...

```ruby 
module Events
  module SumEvent
    module ClassMethods
      def on_sum_event(func=nil, &block)
        return @sum_event_handlers unless func || block
          
        @sum_event_handlers ||= []
        @sum_event_handlers << (func ? method(func) : block)
      end
      
      def invoke_sum_event_handlers(instance)
        return unless @sum_event_handlers.kind_of? Array
        
        @sum_event_handlers.each { |h| h.call(instance) }
      end
    end
    
    def self.included(parent)
      parent.extend(ClassMethods)
    end
    
    def trigger_sum_event
      self.class.invoke_sum_event_handlers(self)
    end
  end
end
```

Notice that this is a module, not an actual object.  The event is really just behavior we are decorating on the including class.  While there isn't much code here, this is some fairly advanced Ruby.  Let's break it down.

First we have a couple class methods, *on_sum_event* and *invoke_sum_event_handlers*.  

*on_sum_event* is used by the including class as a way to register an event handler.  Note its signature: 

```ruby
def on_sum_event(func=nil, &block)
```

So it registers either a function or a block, which we see at:

```ruby
@sum_event_handlers << (func ? method(func) : block)
```

How does this work?  Well, first thing we see is we're giving preference to a function if one is passed in, and then we're calling a method called *method* on it. What on earth is going on?

Ruby has something called a [Method object](http://ruby-doc.org/core-2.2.0/Method.html).  This allows us to pass a method around like data to be called at a later time.  Note that Method's parent is Object, so this feature applies to just about any common object we might encounter or create ourselves.

If a function is not present but we have a block, then we simply pass back a block to our collection of handlers.

If we happen to register both a method and a block to our collection, what do we see?  Jumping ahead a bit, we will be defining the following handlers in MyPoro:

```ruby
on_sum_event { |instance| puts "Your sum is: #{instance.sum}" }
on_sum_event :sum_warning_handler
```

Simply calling *MyPoro* executes the code for the class, which is where the two handlers above are wired up.  Calling *MyPoro.on_sum_event* without a method or block simply returns the handlers we already have.

Inspecting in irb, we see:

```ruby
$ irb
irb(main):001:0> require './my_poro'
=> true
irb(main):002:0> MyPoro.on_sum_event
=> [#<Proc:0x007feeea8ef680@/my_poro.rb:15>, #<Method: MyPoro.sum_warning_handler>]
irb(main):003:0> 
```

So there we have it, our blocks are Proc objects and our methods are Method objects.  We can invoke either with the same syntax:

```ruby 
my_proc_object.call(params)
my_method_object.call(params)
```

With that out of the way, *invoke_sum_event_handlers* should make more sense:

```ruby
def invoke_sum_event_handlers(instance)
  return unless @sum_event_handlers.kind_of? Array
  
  @sum_event_handlers.each { |h| h.call(instance) }
end
```

What is **instance** that is passed in?  Well, it is the object instance the event is called on.  Remember this is a class method - the handlers are defined at the class level and they apply to all instances, but they are *triggered by an instance*.  As such, we can and should make the instance available to each handler.

This brings us to the final bit of behavior in our SumEvent, which is our instance-level trigger method:

```ruby
def trigger_sum_event
  self.class.invoke_sum_event_handlers(self)
end
```

As this is an instance method, we have to call into the class itself to access *invoke_sum_event_handlers*. Then we pass in *self* so the handlers have access to the object instance.

# Bringing it All Together
Now we have our SumEvent ready to use, we include it in MyPoro and move out our code for printing out the sum to the console and reporting any warnings to standard error:

```ruby 
class MyPoro
  include Events::SumEvent

  def self.sum_warning_handler(instance)
    if instance.sum > MAX_SUM
      $stderr.puts "Warning: sum is greater than max of #{MAX_SUM}. Let's not get crazy folks!"
    end
  end
  
  on_sum_event { |instance| puts "Your sum is: #{instance.sum}" }
  on_sum_event :sum_warning_handler
  
  # other code omitted
end
```

And we can update *calculate_sum* to trigger the event:

```ruby
def calculate_sum(lower, upper)
  @sum = (lower..upper).to_a.reduce(:+)
    
  # Broadcast event
  trigger_sum_event
  
  @sum
end
```

Nice and clean.  Now *calculate_sum* does not care about what to do when a sum has been calculated, it simply broadcasts an event, which in turn invokes the registered event handlers.

# Caveats
The handlers here are synchronous, and because of that there can be performance implications if certain operations are egregiously blocking (ie. network latency when logging to a 3rd party service).  We started this post with a comparison to ActiveRecord hooks - those are synchronous for a reason, insofar that certain actions within a hook may need to stop forward motion of the action.  You can always background the meat of a handler at any time (see: [Just Background It!](/just-background-it)), or you could write an Async Event.

As well the code for an event here is pretty generic.  If you started adding other events, you can see how there would be a fair amount of duplication.  While you could add helper funcs to DRY it up, meta-programming is the better answer here, perhaps creating a simple DSL to spec out events.  I'll save that for a future post.

# Finally

Feel free to check out [an example of this in action](https://github.com/brandall10/just_event_it_demo), and feel free to reach out with any comments!
