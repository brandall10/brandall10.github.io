---
layout: post
title:  "Just Background It!"
date:   2016-04-11
categories: rails
permalink: /just-background-it
---

# Background What?
Most Rails developers are aware that backgrounding mailers is a best practice.  After all, the ability to do so is baked right into ActionMailer's API.  Aside from that, it's common to schedule tasks to run outside of any actual web request (ie. database cleanup). For the purpose of the post let's just focus on things which affect request response time.
 
So what else then?

First off, if the reason to background mailers isn't obvious, perhaps it's best to start with a 
simple premise: 

> You should background high-latency activities which are not necessary to complete a request.

In broad strokes, addressing IO-bound concerns is going to bear the most fruit.

* #### __API calls__
API calls are oftentimes the best place to look for performance gains. Not only are they usually high 
latency (on the order of hundreds of milliseconds or more), but they tend to be highly variable in 
performance.  You're beholden to the load and locality of a 3rd-party, act accordingly. 

* #### __Large Database Transactions__
Even if you're following best practices on the DB end of things, sometimes you just can't get away from 
the fact that you have to book a lot of data.  And this can be compounded when working on a system at 
high load.  You can always save off the minimal amount of data needed for a particular request, and if 
any update/cleanup activities are not necessary nor can be rectified by additional user input, go ahead 
and background it.

* #### __Large Units of Work__
Sometimes you have a fair amount of work which needs to be performed as a unit, but that work does not have 
to constrain further user movement in the application.  Think of a user registration: after the data is
validated and a payment has been processed (in the case of a paid service), the remaining activities 
can be backgrounded allowing control to be returned to the user.

The third bullet above is a meta concern of the first two, but it's worth mentioning as we should only 
really be backgrounding one thing in a sequence.  In other words, there rarely is a need to background an 
API call within a particular task which has already been backgrounded itself.  The main point is we're 
freeing up the main thread so the request itself finishes in the least time possible with a minimal 
amount of additional complexity. 

# Human vs. Machine, Fight!
It's helpful to think about the context of the request.  Of course a user request is more important that
say, processing a webhook, right?  Not so fast.

Let's consider a scenario which exists in a project I'm currently working on - signing up a user for a 
recurring subscription plan through Stripe.  Many things happen during this process; a whole
lotta API calls, db transactions, and webhooks are processed when we're all said and done. Let's examine
what happens in each part of the process.

#### __User Request__
The recommended Stripe signup approach is as such:

1. Client-side: submit payment info to Stripe, get a token in return
2. Server-side: using the token provided by the client, submit to Stripe to create an account and process the initial payment.

The main reason for this division of labor is to obviate concerns about PCI compliance as we are able to keep
all payment info off the server.  Both steps are necessary to move a user forward in the system.

The nice thing about step #2 is Stripe packages up things for us to some degree.  With a new subscription we
get a customer object, a subscription object, and a card object.  What we don't get and still may require
is the first invoice with the completed charge attached.  That requires another request to Stripe.  

There's also a monkey wrench on top of this: it is quite possible you may want to attach metadata to these
Stripe objects; for each object you do so that is an additional request.  In the case of my current project,
we do all of this, which in total ends up being seven calls to Stripe.  When doing this implementation we
were seeing responses from Stripe anywhere from ~200ms to ~1.5 seconds per request.  In other words, setting
metadata in the main thread became unreasonably expensive.  When backgrounding that work, it took our user 
signup from being in the 5-10 second area to ~1-3 seconds.  Crucial stuff for successfully onboarding new users.

#### __Webhook__
Stripe webhooks are highly granular - just one type of activity on one object - and as such take relatively little work 
to process.  If you persist everything locally you may only have the constraint of attaching metadata to any new 
objects created; so just one call to Stripe.  But if you don't and say receive a new invoice and need data on the
subscription it is attached to, that is a call to Stripe.  And if you need additional data on the customer account
that is yet another call to Stripe.  

For a plain vanilla recurring subscription payment the system receives a new invoice and a new charge object every payment period for every user.  This doesn't sound like much, and initially it isn't, but what happens on a site with many users?

Thinking this through, we can imagine this could be a bit problematic during times of high activity.  You don't really
have control of the time of day when Stripe processes the payment, and if you have thousands of users you can
easily have hundreds of automated transactions happening on a given day.  Each webhook takes up a worker while 
it processes, and the more time it takes to turnaround a webhook the more requests will be queued during 
times of high activity.  You can always beef up your servers a bit to account for the extra overhead and that's 
fine, but at the the same time it's easy to background what is essentially unnecessary work - no user cares about 
this webhook activity in a direct sense nor should your sysadmin.  And this is just the tip of the iceberg as Stripe sends *alot* of webhooks.  

The only thing Stripe cares about when it sends a webhook is whether or not it has been successfully processed.  If it gets back a 200 then it is happy and stops sending it.  If it gets back something other than 200 or times out it continues to send it hourly.  That's it in a nutshell - as long as you have what you need to successfully process it, just background that activity return 200.

Don't let a webhook place undue stress on your server throughput - Just background it!

# Rolling Up Our Sleeves...
Alright, so now we have some idea of the problem domain, just how do we go about backgrounding?  First off,
let's introduce some concepts we'll use throughout the rest of this article: 

* Service Objects - A PORO (Plain Ol' Ruby Object) to encapsulate behavior in a standardized interface for backgrounding
* ActiveJob - A generic Rails interface for various job queueing backends
* Mixin - To add backgrounding behavior to a Service Object

Most intermediate Rails developers are probably familiar with these concepts so we won't be going into 
too much detail on any of these; suffice to say there are numerous blog posts you can refer to with a quick Google. 

The key idea from all of this is we want to be as flexible as possible.  We want backgrounding to be *easy*; there really
shouldn't be much in the way of excuses against doing it (ie. developer / team experience, management pushback).  Best of all, we want be elegant and do it with minimal code; no need to make a mess of things to be performant.  

## Service Objects
A Service Object is a concept which has caught on in recent years as a way to manage complexity inside a Rails application.  It arguably deviates from The Rails Way (as prescribed by DHH and other traditionalists) as it's not a conventional 
construct provided by the Rails framework.  Detractors of its use typically say there is some inherent design 
issue in the application, that its a bandaid of sorts.  Personally, I'm a proponent but only conditionally use it
in these conditions:

1. A way to refactor out logic from overly fat models
2. A way to refactor out logic from frothy controllers which I don't want to put into models or branch out into other controllers
3. Any logic which has cross-cutting concerns (ie. which doesn't really belong to any *one* particular model or controller)
4. A wrapper around API calls

Generally speaking, it's something you're going to think about using on any application which has gone beyond the trivial
stage in size and scope, and it's a great thing to use if you're trying to figure out how to collect work which needs to be backgrounded.

So what does a Service Object look like?  Just imagine a class which only handles initialization with one public method
to execute the work:

```ruby
class MyApiOperation
  def initialize(params)
    @message = @params
  end
  
  def call
    response = Api.call(@params)
    
    transformed_response = transform_response(response)
    
    ApiResultsModel.create!(results: transformed_response) 
  end
  
  private
  def transform_response(response)
    # do stuff to response and return results
  end
end
```

And in practice you would use it as such:

``` ruby
# Like a method call
MyApiOperation.new(params).call

# Passed around as an object with delayed invocation
api_operation = MyApiOperation.new(params)
# do stuff
api_operation.call
```

The advantages of using a Service Object over a simple method are:

1. Being a class, it provides the advantages of encapsulation, composition, inheritance, etc, which are helpful for managing complexity
2. It defines an interface in your app

Point #1 is a pretty important one.  Your Service Objects can get frothy, and as such a complex operation may be comprised of many Service Objects.  Ideally areas of high churn should be mostly relegated to Service Objects, not your models and controllers, and if properly designed are easy to extend and refactor.

For the remainder of this post let's just think of a Service Object as a place to put some operation we want to background.

# Active Job
Active Job is a relatively new thing to the Rails world, having been introduced in Rails 4.2.  In short it's a generic interface for job queuing backends, so for instance switching from Sidekiq to Resque becomes a configuration detail rather than an implementation detail. 

It does have a limitation though in that the only complex types it understands are ActiveRecord objects.  It also makes no assumptions about the work you want to queue, so we need to get our hands dirty a bit.

# Running a Service Object with ActiveJob
First off, let's create an ActiveJob object for running any Service Object:

```ruby
class ServiceJob < ActiveJob::Base
  queue_as :default
  
  def perform(service, params)
    ActiveRecord::Base.connection_pool.with_connection do
      begin
        service.constantize.new(params).call
      rescue => e
        logger.error "Service #{service} threw an exception of"\
                     "type '#{e.class}' with message: '#{e.message}'"
      end
    end
  end
end
```

Note how we're checking out a connection from the Active Record connection pool.  While this isn't going to be necessary for all operations, we can imagine that performing database IO is common enough for a Service Object that it's better safe than sorry.   

For the actual Service Object call we can see the power of using this type of interface - we can marshall any object defined in the application, pass along its params, and call it.

With this, we can background any Service Object with ServiceJob like so:

```ruby
ServiceJob.perform_later(MyApiOperation.to_s, params)
```

That works.  But it's not too pretty, and quite frankly a bit dangerous.  A junior developer might be inspired to create a table of String mappings to Service Objects in the system, introduce a typo, and boom.

Can we make this a bit better?  Let's give it a shot with another layer of abstraction which can be attached to any Service Object with a mixin: 

```ruby
module Backgroundable
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def execute_in_background(params)
      ServiceJob.perform_later(self.to_s, params)
    end
  end
end
```

Going back to MyApiOperation, we can add this behavior like so:

```ruby
class MyApiOperation
  include Backgroundable
  ...  
end
```

Pulling this all together, now we can background a MyApiOperation call like so:

```ruby
MyApiOperation.execute_in_background(params)
```

Slick!  Now with a fairly small amount of code we have a means to take any unit of work and attach the capability to background it in our application with any job queueing backend of our choosing. 

If you want to see this in action check out [a demo app](https://github.com/brandall10/just_background_it_demo), and feel free to leave any comments or questions!
