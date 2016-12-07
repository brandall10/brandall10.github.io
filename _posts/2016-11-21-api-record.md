---
layout: post
title:  "ActiveRecord: Automagically Marshall an API Response"
date:   2016-11-21
categories: rails api
permalink: /api-record
---

Have you ever had to sync a record from a third party to a local db?  I have -- in all sorts of systems/languages -- and that type of marshalling can be a huge source of grief, esp. if your third party is not a good citizen when it comes to versioning an API and managing breaking changes.

Fortunately ruby is here to help. At least at an entity-level, we can:

1. Map a json object's attributes directly to table-level field names
2. Remap any fields from a json object to ones defined in your table
3. Track any remaining invalid fields
4. Give the option to permit invalid fields so superificial API changes don't break production

Alright, let's roll up our sleeves and attack these features one-by-one.  

## Mapping the API response directly to an ActiveRecord object
First, we will create a concern called ApiRecord.

```ruby
module ApiRecord
  extend ActiveSupport::Concern

  def initialize(attributes=nil)
    if api_attrs = attributes.try(:[], :_api_object)
      attributes.merge!(api_attrs).delete(:_api_object)

      attributes.symbolize_keys!
    end

    super
  end
end
```
We need to override initialize to work our magic. As a convention, we pass in an _api_object as an attribute. This is the json data source we have been given from a third party.  We merge it into our current set of attributes and then remove it as it is not an actual attribute itself. Finally, we call symbolize_keys!, an ActiveSupport helper,  on our attributes collection to normalize our keyspace to symbols.  Easy enough.

## Remapping any fields 
In the case of fields from your service provider being altered, or perhaps just a desire to store data with different field names locally, we should offer the ability to remap any fields.  Let's create a feature called 'has_api_attr_mappings'.

```ruby
module ApiRecord
  extend ActiveSupport::Concern

  class_methods do
    def has_api_attr_mappings(api_attr_mappings={})
      class_attribute :api_attr_mappings
      self.api_attr_mappings = api_attr_mappings
    end
  end

  def initialize(attributes=nil)
    if api_attrs = attributes.try(:[], :_api_object)
      # prior code omitted
      
      if respond_to? :api_attr_mappings
        attributes = remap_api_attrs attributes
      end
    end

    super
  end

  private
  def remap_api_attrs(attributes)
    attributes.map{ |k, v| [api_attr_mappings[k]||k, v] }.to_h.reject{|k| k == :nil}
  end
end
```

This bit is certainly more involved.  Let's break it down.  

First, we have a class level method which allows us to enable the feature called 'has_api_attr_mappings'.  Calling this defines a class attribute called api_attr_mappings which the defined hash is passed to.

Next, you can see our changes to initialize.  Then we check to see if api_attr_mappings is defined.  If so, we know we have defined this feature, so we proceed to remap the attributes.

Lastly, we have a private helper which does the heavy lifting. It's small but mighty, taking the mapped keyspace and remapping each key which has a map to the mapped key, otherwise retaining the original mapping.  As a final touch, it has the ability to scrub anything we map to :nil -- there may be data from your service provider that you just don't care about and want to ignore.

## Track invalid fields
If there are any remaining keys which can't be accounted for in direct or indirect mappings, we still want allow for the object to be created while tracking these invalid attributes.  By default we want to ensure the object is invalid. 

```ruby
module ApiRecord
  included do
    attr_accessor :invalid_api_attrs
    validates :invalid_api_attrs, absence: { message: "%{value}" }
  end

  def initialize(attributes=nil)
    if api_attrs = attributes.try(:[], :_api_object)
      # prior code ommitted

      strip_invalid_api_attrs! attributes
    end

    super
  end

  private
  def strip_invalid_api_attrs!(attributes)
    self.invalid_api_attrs = []
    columns = self.class.column_names.map(&:to_sym)
    
    attributes.select! { |a| columns.include?(a) ? true : self.invalid_api_attrs << a && false }
  end
```

We add an included block to create the invalid_api_attrs attribute and an absence validation on it. Then we strip the attributes in our initialize with a private method which does the heavy lifting.  

Taking a look at the last part, we get a list of the columns in symbol form (remember our call to symbolize_kays!).  Then we iterate through our list of attributes, which by now should have been remapped where specified.  Using a select we indicate which attributes we want to retain based on which match to our database.  Any misses we have are saved off to our invalid_api_attrs collection and removed from the attributes list.  Now with this, we should have a fully fleshed out (to the best of our abiilty) AR instance.

## Permit invalid fields
This last part could be seen as a bit controversial, but as mentioned in the intro, we can not always rely on a third party service to be a good citizen.  There are times when non-breaking changes will be added to a live API (ie. new fields), and hopefully if they're breaking it's kept to a superficial level. We can still track the invalid fields in the invalid fields collection while opting out of the absence validation when specified. Let's make that adjustment.

```ruby
module ApiRecord
  included do
      class_attribute :allows_invalid_api_attrs
      self.allows_invalid_api_attrs = false

      attr_accessor :invalid_api_attrs
      validates :invalid_api_attrs, absence: { message: "%{value}" }, unless: "allows_invalid_api_attrs?"
    end
  end

  class_methods do
    # prior code omitted

    def permit_invalid_api_attrs
      self.allows_invalid_api_attrs = true
    end 
  end
end
```
We've added permit_invalid_api_attrs as a convenience helper.  It provides a nice directive to set the class attribute allows_invalid_api_attrs behind the scenes.  We instantiate this to false, which is our desired default behavior, and add the check to the validation.

## Putting it all together

The code from the previous sections, stitched together w/ comments:

```ruby
require 'active_support/concern'

module ApiRecord
  extend ActiveSupport::Concern

  included do
    class_attribute :allows_invalid_api_attrs
    self.allows_invalid_api_attrs = false

    attr_accessor :invalid_api_attrs
    validates :invalid_api_attrs, absence: { message: "%{value}" }, unless: "allows_invalid_api_attrs?"
  end

  # Define features at the class level
  class_methods do
    def has_api_attr_mappings(api_attr_mappings={})
      class_attribute :api_attr_mappings
      self.api_attr_mappings = api_attr_mappings
    end
    
    def permit_invalid_api_attrs
      self.allows_invalid_api_attrs = true
    end 
  end

  def initialize(attributes=nil)
    # Only apply ApiRecord features if an _api_object is passed in
    if api_attrs = attributes.try(:[], :_api_object)
      # By default merge in all attribs and delete source object
      attributes.merge!(api_attrs).delete(:_api_object)

      # Normalize keys to all symbols
      attributes.symbolize_keys!
      
      # Remap any attributes if defined
      if respond_to? :api_attr_mappings
        attributes = remap_api_attrs attributes
      end
      
      # Permit and log invalid attributes if allowed
      strip_invalid_api_attrs! attributes
    end
   
    # Now all work is done, invoke parent
    super
  end

  private
  def strip_invalid_api_attrs!(attributes)
    self.invalid_api_attrs = []
    columns = self.class.column_names.map(&:to_sym)
    
    attributes.select! { |a| columns.include?(a) ? true : self.invalid_api_attrs << a && false }
  end
  
  def remap_api_attrs(attributes)
    attributes.map{ |k, v| [api_attr_mappings[k]||k, v] }.to_h.reject{|k| k == :nil}
  end
end
```

## In Use

To see how this is actually used in a model, imagine we have an ApiAccount class, which is course syncs account info from a third party API service.

```ruby
class ApiAccount < ApplicationRecord
  include ApiRecord

  permit_invalid_api_attrs

  has_api_attr_mappings :account => :api_account_id,
                        :status => :api_status,
                        :history => :nil

end
```
We see the developer has chosen to remap a few fields.  The account field from the service is actually the key for the record, but this field name is not congruent with the naming scheme used in the local db.  As well, we see a status field come in, but there already is a status field on the record, so it is renamed to api_status to prevent a collision.  Finally, a large embedded json field called history is sent to :nil, which means it is discarded as the client does not care about it.

Now let's change this a bit and remove ':history => :nil'.  Now when the record is instantiated, history will still be scrubbed from the record, but if you check record.invalid_api_attrs you will see it is '[:history]'.  To fix this issue, the developer has two choices -- add it back to the api_attr_mappings, or add the field to the database.

I've provided [some sample code](https://github.com/brandall10/api_record) if you would like to see this in action.  Note that this application does not even have a hard model, this concern is tested with an on-the-fly model using the handy [temping gem](https://github.com/jpignata/temping).  I'll go into more detail on why I'm doing this in a future blogpost.

