# Caching Strategies for Ultra-High Performance in Ruby on Rails

In this post, we will discuss caches, caching strategies, and cache stores. We will also explain how to expire or invalidate caches and discuss the benefits caches provide for our projects.

## What is caching

Caching in web development refers to the process of storing and reusing previously fetched or computed data to reduce the time it takes to retrieve or generate that data again. The goal of caching is to improve performance and efficiency by serving content more quickly, reducing server load, and minimizing the need to repeat resource-intensive operations.

## Caching benefits

- Better performance
- Handle more users
- Reduce resource consumption

## Caching disadvantages / drawbacks

- Extra work is needed to implement caching strategies
- Need to handle cache expiration
- Need to monitor cache correctness and metrics

## Page caching

Page caching involves caching entire HTML pages. This is suitable for pages that don't rely on dynamic data and can be pre-rendered.
To enable page caching, you can use the caches_page method in your controller:

```ruby
class HomeController < ApplicationController
  caches_page :index
  
  def index
    # Controller logic
  end
end
```

This will automatically cache the entire HTML page for the index action.
Page caching was removed from Rails 4, it can still be used with the following gem.
https://github.com/rails/actionpack-page_caching


## Action caching

Action caching is needed for actions that use filters, such as authentication and validation. When action caching is used, the request hits the Rails stack, goes through the filters, and then the cache is served. Action caching was removed from Rails 4. To use it, the following gem is needed.

https://github.com/rails/actionpack-action_caching

```ruby
class ListsController < ApplicationController
  before_action :authenticate, except: :public


  caches_page   :public
  caches_action :index, :show
end
```

This code snippet shows three methods. The public method uses page caching because it does not need any filters. On the other hand, the index and show methods use action cache because the user needs to be authenticated through the before_action filter, which calls the authenticate method.

## Fragment caching

It allows to cache pieces of a view, it’s especially helpful to cache partials or components that don’t change too much or too often, like headers, footers, sidebars, etc. To use fragment caching, you can wrap a portion of your view with the cache helper method. For example:

```ruby
<% @products.each do |product| %>
  <% cache product do %>
    <%= render product %>
  <% end %>
<% end %>
```

## Russian Doll caching
This cache allows you to nest cache fragments inside other cache fragments. If you have a hierarchy of models like Article, Comment, and User, you might cache fragments at each level.

The key aspect of Russian Doll Caching is the ability to expire or invalidate only the relevant cache fragments when a record is updated. This is typically done by including a version identifier or timestamp in the cache key.

When a record is updated, its associated cache fragments are invalidated, and the next request triggers the regeneration of only the affected fragments.
Russian Doll Caching helps in minimizing the amount of unnecessary caching and regeneration of content. It allows you to isolate the impact of changes to specific parts of your application.

Suppose you have a view that displays an article and its associated comments. You might cache the article and comment fragments separately. If a comment is added or edited, only the associated comment fragment needs to be updated, not the entire article fragment.

```ruby
# In the view for displaying an article
<% cache @article do %>
  <!-- Article content -->


  <% @article.comments.each do |comment| %>
    <% cache comment do %>
      <!-- Comment content -->
    <% end %>
  <% end %>
<% end %>
```

In this example, when an article **updated_at** changes, its cache will be invalidated, but the comments **updated_at** will remain the same, and stale content will be served. To avoid that, the touch method can be used in the following way:

```ruby
class Article < ApplicationRecord
  has_many :comments
end


class Comment < ApplicationRecord
  belongs_to :article, touch: true
end
```

Now everytime that the article **updated_at** attribute is updated, expiring it’s cache, the same will happen for the comments.

## Low-level caching

Low-level caching should be used when you need more fine-grained control over caching, such as caching a particular value or query result instead of caching view fragments. 
Serializable information is handled extremely well by Rails' caching mechanism.

The **Rails.cache.fetch** method is usually used to implement low-level caching. This is because fetch can read and write to the cache, depending on the parameters given.

When given **only a single argument**, the key is fetched, and the value from the cache is returned. If a **block is passed**, that block will be executed in the event of a cache miss. The return value of the block will be written to the cache under the given cache key, and that return value will be returned. In case of a cache hit, the cached value will be returned without executing the block.

```ruby
class Product < ApplicationRecord
  def competing_price
    Rails.cache.fetch("#{cache_key_with_version}/competing_price", expires_in: 10.hours) do
      Competitor::API.find_price(id)
    end
  end
end
```

**cache_key_with_version** method generates a string key based on a model’s class name, id, and updated_at attributes. This key will be the cache key.

This is a common convention and has the benefit of invalidating the cache whenever the product is updated. In general, when you use low-level caching, you need to generate a cache key.

## Cache Stores
Ruby on Rails supports several cache stores out of the box. You can set your application cache store using this configuration option: config.cache_store. This option normally belongs to the environment files, /config/environments/development.rb | production.rb | test.rb.


### Memory Store

**ActiveSupport::Cache::MemoryStore**, is a simple cache store that stores data in the application's memory. This is suitable for small-scale applications or development environments, but it is not suitable for production environments with multiple server instances.

```config.cache_store = :memory_store, { size: 64.megabytes }```

Is important to know that If you're running multiple Ruby on Rails server processes, for example, if you're using Phusion Passenger or puma clustered mode, then your Rails server process instances won't be able to share cache data with each other.

### File Store

**ActiveSupport::Cache::FileStore**, stores cached data in the file system. It is more scalable than the memory store and is suitable for small to medium-sized applications. Each server instance has its own cache file.

```config.cache_store = :file_store, "/path/to/cache/directory"```

With File Store, multiple server processes on the same host can share a cache. This is the default cache store implementation; the cache files are going to be created at **"#{root}/tmp/cache/"**.
The File store cache files can grow indefinitely, so it’s important to clear the cache on a regular basis.

### Null Store

**ActiveSupport::Cache::NullStore**, is scoped to each web request, and clears stored values at the end of a request. Is mainly used for development and test environments. It is helpful when our code uses Rails.cache directly and the development process changes are not being reflected on our app; in those cases, if we set the cache store to null store, no cache is going to be used.

```config.cache_store = :null_store```

There is a command that also helps us to test cache in the development environment, rails dev:cache:

```
$ bin/rails dev:cache
Development mode is now being cached.
$ bin/rails dev:cache
Development mode is no longer being cached.
```

### Memcache

**ActiveSupport::Cache::MemCacheStore**, a distributed memory object caching system. This allows multiple server instances to share a common cache. Memcached is often used in production environments with multiple application server instances. When initializing the cache, you should specify the addresses for all memcached servers in your cluster, or set  the MEMCACHE_SERVERS environment variable.

```config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com"```

If the server addresses are not provided and the env variable is not set, memcached will be assumed to be running on localhost on the default port (127.0.0.1:11211). This is not recommended for production environments.


### Redis

**ActiveSupport::Cache::RedisCacheStore**, uses Redis, an advanced key-value store. Redis is commonly used for distributed caching and is highly scalable. It allows multiple application server instances to share a common cache. Important considerations:

- Redis support for automatic eviction when it reaches max memory (it releases/deletes the cache that is less frequently used)
- Redis doesn't expire keys by default; use a dedicated Redis cache server
- For a cache-only Redis server, set maxmemory-policy to one of the variants of allkeys
- The cache store will attempt to reconnect to Redis once if the connection fails during a request.

To use this cache store, first, we need to install Redis; for that, we need to add the gem to the Gemfile.

```ruby
gem 'redis'
```

Then it can be used and configured in the environment files like the other stores.

```config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }```

RedisCacheStore also supports configuration parameters like, connect_timeout, read_timeout, write_timeout, reconnect_attempts and error_handler, this last one allows sending exceptions to other services, like Sentry.

## Cache expiration

It’s essential to remember that the cache mechanisms need expiration or invalidation rules or mechanisms, otherwise stale data can be served. Some cache mechanisms have a default expiration time but it’s better to declare when cached data is going to expire explicitly. How the expiration time is set depends on the cache mechanism.

For Page caching, the cache files need to be deleted.

For Action caching the expiration, can be specified using the following syntax:

```ruby caches_action :archived, expires_in: 1.hour```


For Fragment caching, the expiration can be handled like this:
```ruby
  <% cache product, expires_in: 1.hour do %>
    …
  <% end %>
```

For low-level caching, it can be set using:
```ruby
    Rails.cache.fetch("#{cache_key_with_version}/states", expires_in: 24.hours) do
      …
    end
```

In addition to the cache mechanism expiration options, most cache stores have ways to set or configure expiration times. That configuration depends completely on the cache store. We will not cover those configuration options here, but the stores have proper documentation that can be referenced.
 
## Performance Improvement

You may be wondering how much cache really helps improve your app performance and reduce page load time. Because we have not talked about any numbers yet, let's suppose that we have a simple application, tutorial level, and we use page cache for the About section or any other section that is mainly static and can be cached easily. The first time that you load the page after executing the server, it could take around 200 to 500 ms, but the following request to that same page will take 20 to 50 ms approximately. **The page loads 10 times faster**. In a production application with thousands or millions of users, it will be harder to find what to cache or how to improve an already on-place cache mechanism, but even a 10% or even 5% page load time would be noticeable for the users, resulting in a better user experience. This also reduces the server's workload.

The performance improvement cache mechanisms can provide depends on many factors, such as what resources you are caching, which cache store you are using, cache eviction, latency when communicating to the cache servers, etc. It’s important to measure load times before and after cache implementation to make sure that the cache is actually improving our application load times.

## Conclusion

Caching strategies should be chosen based on the specific requirements of your application. Additionally, we need to consider cache expiration, cache key strategies, and potential issues related to content staleness when implementing caching in Ruby on Rails applications. The cache store is a key component of the cache implementation, it can be chosen based on many factors, flexibility (configuration options), level of scalability needed by the application, whether the application runs on a single server or in a distributed environment, and the desired balance between simplicity and performance.

Cache implementation is a must-know skill that allows us to improve our application performance and greatly enhance our application user experience.

