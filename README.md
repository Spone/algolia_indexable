# AlgoliaIndexable

Easily index your ActiveRecord models in Algolia.

## Advantages

- use the latest Algolia Ruby client directly in your Rails app
- no DSL, just plain Ruby objects, modules and methods with single responsibility
- small feature footprint
- sane defaults
- advanced behaviors are easy to implement:
  - [index one model in multiple indexes](#connect-your-models)
  - [split one model in multiple records](#create-your-serializers)
  - [disable indexing of a record based on custom logic](#disable-indexing-for-a-record)

## Installation
Add the gem to your application's Gemfile with:

```bash
$ bundle add algolia_indexable
```

Setup your Algolia config by adding credentials:

    bin/rails credentials:edit

```
algolia:
  application_id: 
  admin_api_key: 
  search_api_key: 
```

Alternatively, you can add the following ENV variables. They will be used if no credential is detected.

```
ALGOLIA_APP_ID=
ALGOLIA_ADMIN_API_KEY=
ALGOLIA_PUBLIC_API_KEY=
```

There are two additional settings that can be defined with ENV variables:

```
ALGOLIA_ENV=
ALGOLIA_DISABLE_INDEXING=
```

- `ALGOLIA_ENV` is used as a suffix for index names. If multiple developers are working on the project at the same time, they can set it as their first name, so their developments environments are separated (else the development Algolia indexes will collide). If not defined, it defaults to `Rails.env`.
- `ALGOLIA_DISABLE_INDEXING` can be set to `true` to disable all indexing operations in the current environment. If not defined, it defaults to `Rails.env.test?` so indexing is disabled in the test environement by default.

## Usage

### Create your indexes

Each Algolia index has a matching Ruby class, inheriting from `ApplicationIndex`.

Use the `settings` class method to define your settings for the index.

```rb
# app/indexes/product_index.rb
class ProductIndex < ApplicationIndex
  def self.settings
    {
      searchableAttributes: %w[name description category],
      attributesForFaceting: %w[category],
    }
  end
end
```

They will be set whenever you reindex your model, using the Ruby client (`#set_settings`)[https://www.algolia.com/doc/api-reference/api-methods/set-settings/#simple-set-settings] method.

### Create your serializers

Before indexing your record, it will be serialized based on the following rules, and your custom Serializer.

The following default attributes will always be present in your indexed record:

```rb
{
  objectID: algolia_object_id, # e.g. "Product#514" (or "Product#514/1" if you split your model)
  model_name: model_name,      # e.g. "Product"
  model_id: algolia_model_id   # e.g. "Product#514"
}
```

You can override `#algolia_object_tid` and/or `#algolia_object_id` in your model to customize these values if needed.

Then, your serializers receive an ActiveRecord instance and serialize it for indexing.

They are located in `app/serializers/indexing`.

They must have at least one method: `#record`, that returns a Hash.

If you want to split your ActiveRecord into multiple Algolia records, you must provide another method: `#records`, that returns an Array.

```rb
# app/serializers/indexing/product_serializer.rb
module Indexing
  class ProductSerializer
    def initialize(product)
      @product = product
    end

    def record
      {
        name: @product.name,
        description: @product.description,
        category: @product.category,
        created_at: @employer.created_at.to_i
      }
    end

    # If you want to split your ActiveRecord into several Algolia records
    # you must specify how the split is performed:
    def records
      description_parts = @product.description.split("/n")

      description_parts.map do |part|
        record.merge(
          description: part,
        )
      end
    end
  end
end
```

### Connect your models

To make a model indexable, you have to include the `AlgoliaIndexable` concern, then call `#algolia_index_in` with the Index class and the Serializer class to use.

```rb
# app/models/product.rb
class Product < ApplicationRecord
  include AlgoliaIndexable
  algolia_index_in ProductIndex, serializer: Indexing::ProductSerializer
  algolia_index_in EverythingIndex, serializer: Indexing::EmployerEverythingSerializer
end
```

Alternatively, for simpler models, you can also only create an `#algolia_attributes` method directly on the model, instead of using a serializer:

```rb
# app/models/page.rb
class Page < ApplicationRecord
  include AlgoliaIndexable
  algolia_index_in EverythingIndex

  def algolia_attributes
    {
      title: title,
      description: description
    }
  end
end
```

If you do not specify a Serializer, neither an `#algolia_attributes` method, the default `#attributes` method will be used and all your model's attributes will be indexed.

#### Disable indexing for a record

To use some custom logic to decide if a record should be indexed, you can implement `#algolia_indexable?` on the model:

```rb
# app/models/page.rb
class Page < ApplicationRecord
  # ...

  def algolia_indexable?
    published?
  end
end
```

## Unresolved issues

### How to split into multiple records, with UUID primary key support

The problem is that you cannot do:

```rb
index.delete_by(filters: "model_name:#{model_name} AND model_id=#{algolia_model_id}")
```

if `algolia_model_id` is not a number.

## License
The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
