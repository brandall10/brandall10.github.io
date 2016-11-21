---
layout: post
title:  "ActiveRecord: Automagically Marshall an API Response"
date:   2016-11-20
categories: rails api
permalink: /api-record
---

Have you ever had to sync an entity from a third party to a local db?  I have -- in all sorts of systems/languages -- and that type of marshalling can be a huge source of grief, esp. if your third party is not a good citizen when it comes to versioning an API and managing breaking changes.

Fortunately ruby is here to help. At least at an entity-level, we can:

1. Map a json object's attributes directly to table-level field names
2. Remap any fields from a json object to ones defined in your table
3. Permit and track bad fields, so that API changes can be quickly found during testing

Alright, let's roll up our sleeves and attack these features one-by-one.  

## Mapping the API response directly to an ActiveRecord object
First, we will create a concern called ApiRecord.

```ruby
module ApiRecord
  extend ActiveSupport::Concern

  def initialize(attributes=nil)
    if api_attributes = attributes.try(:[], :_api_object)
      attributes.merge!(api_attributes).delete(:_api_object)
    end

    super
  end
end
```
We need to override initialize to work our magic. As a convention, we pass in an _api_object as an attribute. This is the json data source we have been given from a third party.  We merge it into our current set of attributes and then remove it as it is not an actual attribute itself.  Easy enough.

## Remapping any fields 
In the case of fields from your service provider being altered, or perhaps just a desire to store data with different field names locally, we should offer the ability to remap any fields.  Let's create a feature called 'has_attribute_mappings'.

```ruby
module ApiRecord
  extend ActiveSupport::Concern

  class_methods do
    def has_attribute_mappings(attribute_mappings={})
      class_attribute :attribute_mappings
      self.attribute_mappings = attribute_mappings
    end
  end

  def initialize(attributes=nil)
    if api_attributes = attributes.try(:[], :_api_object)
      attributes.merge!(api_attributes).delete(:_api_object)

      attributes.symbolize_keys!
      
      if respond_to? :attribute_mappings
        attributes = remap_attributes attributes
      end
    end

    super
  end

  private
  def remap_attributes(attributes)
    attributes.map{ |k, v| [attribute_mappings[k]||k, v] }.to_h.reject{|k| k == :nil}
  end
end
```

This bit is certainly more involved.  Let's break it down.  

First, we have a class level method which allows us to enable the feature called 'has_attribute_mappings'.  Calling this defines a class attribute called attribute_mappings which the defined hash is passed to.

Next, you can see our changes to initialize.  We call symbolize_keys! on the attributes collection, which is an ActiveSupport helper on a hash which converts its keys entirely to symbols.  This makes working with the attributes a bit easier.  Then we check to see if attribute_mappings is defined.  If so, we know we have defined this feature, so we proceed to remap the attributes.

Lastly, we have a private helper which does the heavy lifting. It's small but mighty, taking the mapped keyspace and remapping each key which has a map to the mapped key, otherwise retaining the original mapping.  As a final touch, it has the ability to scrub anything we map to :nil -- there may be data from your service provider that you just don't care about and want to ignore.

## Allow and track bad fields
This last part could be seen as controversial. If there are any remaining keys which can't be accounted for in direct or indirect mappings, we can still allow for the object to be created but track the invalid attributes.  Unlike with an invalid model attribute check, we can still persist the data.  

Why would we want to allow this?  As stated in the intro, your service may not be the best citizen or may unknowingly introduce new fields (ie. non-breaking changes).  On a production system, having some flexibility in a domain which you have no control over is certainly helpful. All the same, as the keys are tracked the issues can still be reported.  And then when you do happen to rev an API the changes can readily be caught by your tests prior to deployment.

```ruby
  class_methods do
    # prior code ommitted

    def permit_invalid_attributes
      attr_accessor :invalid_attributes
    end
  end

  def initialize(attributes=nil)
    if api_attributes = attributes.try(:[], :_api_object)
      # prior code ommitted

      if response_to? :invalid_attributes
        strip_invalid_attributes! attributes
      end
    end

    super
  end

  private
  def strip_invalid_attributes!(attributes)
    self.invalid_attributes = []
    columns = self.class.column_names.map(&:to_sym)
    
    attributes.select! do |a| 
      columns.include?(a) ? true : self.invalid_attributes << a && false
    end
  end
```

Much like our remapping work, we have 3 parts to this -- the class-level feature definition, hooking up in intialize, and a private method which does the heavy lifting.  Note that this time 'invalid_attributes' is an instance-level attribute as it is populated by an object.

Taking a look at the last part, we get a list of the columns in symbol form (remember our call to symbolize_kays!).  Then we iterate through our list of attributes, which by now should have been remapped where specified.  Using a select we indicate which attributes we want to retain based on which match to our database.  Any misses we have are saved off to our invalid_attributes collection and removed from the attributes list.  Now with this, we should have a fully fleshed out (to the best of our abiilty) AR instance.

## Putting it all together

The code from the previous sections, stitched together w/ comments:

```ruby
require 'active_support/concern'

module ApiRecord
  extend ActiveSupport::Concern

  # Define features at the class level
  class_methods do
    def has_attribute_mappings(attribute_mappings={})
      class_attribute :attribute_mappings
      self.attribute_mappings = attribute_mappings
    end
    
    def permit_invalid_attributes
      attr_accessor :invalid_attributes
    end 
  end

  def initialize(attributes=nil)
    # Only apply ApiRecord features if an _api_object is passed in
    if api_attributes = attributes.try(:[], :_api_object)
      # By default merge in all attribs and delete source object
      attributes.merge!(api_attributes).delete(:_api_object)

      # Normalize keys to all symbols
      attributes.symbolize_keys!
      
      # Remap any attributes if defined
      if respond_to? :attribute_mappings
        attributes = remap_attributes attributes
      end
      
      # Permit and log invalid attributes if allowed
      if respond_to? :invalid_attributes
        strip_invalid_attributes! attributes
      end
    end
   
    # Now all work is done, invoke parent
    super
  end

  private
  def strip_invalid_attributes!(attributes)
    self.invalid_attributes = []
    columns = self.class.column_names.map(&:to_sym)
    
    attributes.select! do |a| 
      columns.include?(a) ? true : self.invalid_attributes << a && false
    end
  end
  
  def remap_attributes(attributes)
    attributes.map{ |k, v| [attribute_mappings[k]||k, v] }.to_h.reject{|k| k == :nil}
  end
end
```

## In Use

To see how this is actually used in a model, imagine we have an ApiAccount class, which is course syncs account info from a third party API service.

```ruby
class ApiAccount < ApplicationRecord
  include ApiRecord

  permit_invalid_keys

  has_attribute_mappings :account => :api_account_id,
                         :status => :api_status,
                         :history => :nil

end
```
We see the developer has chosen to remap a few fields.  The account field from the service is actually the key for the record, but this field name is not congruent with the naming scheme used in the local db.  As well, we see a status field come in, but there already is a status field on the record, so it is renamed to api_status to prevent a collision.  Finally, a large embedded json field called history is sent to :nil, which means it is discarded as the client does not care about it.

Now let's change this a bit and remove ':history => :nil'.  Now when the record is instantiated, history will still be scrubbed from the record, but if you check record.invalid_attributes you will see it is '[:history]'.  To fix this issue, the developer has two choices -- add it back to the attribute_mappings, or add the field to the database.

I've provided [some sample code](https://github.com/brandall10/api_record) if you would like to see this in action.  Note that this application does not even have a hard model, this concern is tested with an on-the-fly model using the handy [temping gem](https://github.com/jpignata/temping).  I'll go into more detail on why I'm doing this in a future blogpost.

