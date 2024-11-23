---
external: false
draft: false
title: Caching in Ruby on Rails (1)
date: 2024-11-13
---

## Initial Setup

After a long time since I first read it, recently I read once again the official guide about [Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html) and that sparked some curiosity in me to discover a bit more closely what Rails has to offer.

To research this topic, I'll start by creating a new rails project from Rails 8 with all default configuration except Tailwind for the sake of simplicity. This means, I'll be using Solid Cache, Solid Queue and Solid Cable for this article, all of which use SQLite. So, let's get on it.

Let's run the commands to create the new Rails app, and a basic model to have something going.

```bash
 rails --version
Rails 8.0.0

rails new caching --css=tailwind
cd caching
rails g scaffold Post title
rails db:migrate
```

And let's uncomment the last line in our routes:

```ruby
# config/routes.rb
root "posts#index"
```

Now we can start the server and open the browser on http://localhost:3000 to see our site:

```bash
bin/dev
```

![Basic Rails Setup](/images/caching/1/1.png)

[Commit 1 - Setup](https://github.com/divagueame/caching/commit/b12c6fd49e64378e19c093ab2506b95bffbc0e8d)

---

## The most Basic deployment with Kamal

I have my pet project at https://www.facturita.online, so I'll deploy this app on a subdomain using Kamal. I use Cloudflare to manage my DNS, so I created a A Record pointing to my server.
I added a .env file to the root of my project. You will need a account on Docker hub and to create a token which Kamal will use to push the image to their registry. The Rails Master key comes from 'config/master.key'. These values should be kept secret and never shared/commited to your repository.

```
KAMAL_REGISTRY_PASSWORD=dckr_pat_i89mp94MQjMa0KxQW2cmAOvplq8
RAILS_MASTER_KEY=094n921lp9a2ec88341mb514ae49d983
```

and then we can adjust the deploy file Kamal will use to deploy our app:

```
service: caching
image: martinarceteixeira/caching
servers:
  web:
    - caching.facturita.online
proxy:
  ssl: true
  host: caching.facturita.online
registry:
  username: martinarceteixeira
  password:
    - KAMAL_REGISTRY_PASSWORD
builder:
  arch: amd64
env:
  secret:
    - RAILS_MASTER_KEY
```

This is the most basic configuration to get our site live. You can run this command now to setup our site:

```
kamal setup
```

After some minutes, the site is live at: [https://caching.facturita.online/](https://caching.facturita.online/)

[Commit 2 - Setup Kamal](https://github.com/divagueame/caching/commit/0ba240b2928760f26244c5f8968b44bbc1d0ac33)

---

## Config Solid Cache in Development

Rails 8 ships with Solid Cache by default, however it's not enabled on development. So we'll need to make a couple of changes to see our progress on our machine.

Start by running this command to enable Action Controller caching:

```
rails dev:cache
Action Controller caching enabled for development mode.
```

We'll need to configure the cache store to use solid_cache_store instead of default in memory cache:

```
# config/environments/development.rb
#  config.cache_store = :memory_store # Comment out this line
config.cache_store = :solid_cache_store
```

[Commit 3 - Config solid_cache in development](https://github.com/divagueame/caching/commit/dc6164bb4fd8fcec19107bf51bd3beed8eef87fa)

---

## Our very first cache - Fragment Caching

You should definitely have a look at the official Rails guide about [fragment caching](https://guides.rubyonrails.org/caching_with_rails.html#fragment-caching).

Let's say in your template, you have a template that takes a while to compute, like for example this totally natural piece of code:

```ruby
<% 10.times do %>
    <%  sleep 1 %>
    <h1>My beautiful post</h1>
<% end %
```

We can inspect the logs and assert that this is taking around 10 seconds. We could improve that by caching.
![Basic Rails Setup](/images/caching/4/1.png)

Just by wrapping our code in a call to cache, we can speed up all following requests by serving that piece/fragment from our cache instead of computing it on the fly once again:

```ruby
<% cache do %>
  <% 10.times do %>
    <%  sleep 1 %>
    <h1>My beautiful post</h1>
  <% end %>
<% end %>
```

If we have another look at the server logs, we can see what will happen:
![Basic Rails Setup](/images/caching/4/2.png)

1. When the action is triggered, the controller will request the generated hash for that fragment to the cache, in our case, Solid cache.
2. Since it could not find anything, it will write it. So next time this is requested, it will be available.
3. The first request will be processed as usual, in our case it's 10000ms.
4. However, when the next request comes in, the cache is hit and directly served without recomputing the template.
5. So, it will only take 4ms to get that fragment.

Alright, this is the most basic example in a non real scenario, but it's enough to get the point across. You can generate pieces of your template once and keep them until they're needed for another request.

Beware that, even though calling 'cache' on a template without defining a key it's very convenient, beware that you should not have any other unnamed cache block on your template or they keys will conflict. For example, a template like this will not work as expected:

```ruby
  <% cache do %>
    <h1>Dog</h1>
  <% end %>

  <% cache do %>
    <p>Kitty</p>
  <% end %>
```

As it will always render this html:

```html
<h1>Dog</h1>
<h1>Dog</h1>
```

The cache entry is stored by associating a key with a value. The value is the generated template, but we haven't declared any key. Let's have a quick look at [Rails source code](https://github.com/rails/rails/blob/67c6ef2e5957152f00050224281ed4eabaaebd92/actionview/lib/action_view/helpers/cache_helper.rb#L176) to see what that call to 'cache' is doing and how the key is going to be defined:

```ruby
# rails/actionview/lib/action_view/helpers/cache_helper.rb
def cache(name = {}, options = {}, &block)
    if controller.respond_to?(:perform_caching) && controller.perform_caching
        CachingRegistry.track_caching do
            name_options = options.slice(:skip_digest)
            safe_concat(fragment_for(cache_fragment_name(name, **name_options), options, &block))
        end
    else
        yield
    end

    nil
end
```

We can see, a name for this cache entry will be generated by calling: `cache_fragment_name(name, **name_options)`, which then calls `controller.url_for(name).split("://").last` so that if we don't pass any key to our call to 'cache', the generated key will be: `["posts/index:f1e33bba8856ef6fefcc9f67b8635af3", "localhost:3000/posts"]`

We can confirm this by reading the docs:

> [rails/actionview/lib/action_view/helpers/cache_helper.rb](https://github.com/rails/rails/blob/67c6ef2e5957152f00050224281ed4eabaaebd92/actionview/lib/action_view/helpers/cache_helper.rb#L49)
>
> The template digest that’s added to the cache key is computed by taking an MD5 of the contents of the entire template file. This ensures that your caches will automatically expire when you change the template file.
> Template digest

Let's try not to blindly believe this, and let's see it in action.

Let's open a database session:

```bash
bin/rails db

sqlite> PRAGMA database_list; # Let's see what databases we're connected to
0|main|/home/marce/web/rails/caching/storage/development.sqlite3

sqlite> ATTACH DATABASE 'storage/development_cache.sqlite3' AS cache; # Let's connect to the cache database too.
sqlite> .tables
ar_internal_metadata        cache.solid_cache_entries # Here it's our solid_cache table
cache.ar_internal_metadata  posts
cache.schema_migrations     schema_migrations

sqlite> SELECT * FROM solid_cache_entries; # Let's see what we have actually cached:
90|development:views/posts/index:93989b4fc3737f30c7255b32063fa97c/localhost:3000/posts||2024-11-17 09:37:37.905|3017953458958006028|270
```

We can see the row was added to our _solid_cache_entries_ table. However, we can't read the actual HTML string ' <h1>Dog</h1>' that was cached, because it's stored as a binary type. This is its schema:

```ruby
ActiveRecord::Schema[8.0].define(version: 1) do
  create_table "solid_cache_entries", force: :cascade do |t|
    t.binary "key", limit: 1024, null: false
    t.binary "value", limit: 536870912, null: false
    t.datetime "created_at", null: false
    t.integer "key_hash", limit: 8, null: false
    t.integer "byte_size", limit: 4, null: false
    t.index ["byte_size"], name: "index_solid_cache_entries_on_byte_size"
    t.index ["key_hash", "byte_size"], name: "index_solid_cache_entries_on_key_hash_and_byte_size"
    t.index ["key_hash"], name: "index_solid_cache_entries_on_key_hash", unique: true
  end
end

```

Say we have this in our template:

```
<% cache do %>
  <h1>Dog meow</h1>
<% end %>
```

We can then verify the value stored in our cache table from the rails console, and use `Rails.cache.fetch(our key)`. The key we can take it from our previous query on the SQlite (without `development:`), so it will be `views/posts/index:93989b4fc3737f30c7255b32063fa97c/localhost:3000/posts`:

```
rails c
Rails.cache.fetch('views/posts/index:93989b4fc3737f30c7255b32063fa97c/localhost:3000/posts')
  SolidCache::Entry Load (0.1ms)  SELECT "solid_cache_entries"."key", "solid_cache_entries"."value" FROM "solid_cache_entries" WHERE "solid_cache_entries"."key_hash" IN (3017953458958006028) /*application='Caching'*/
=> "    <h1>Dog meow</h1>\n"
```

Sweet! Now can see how the data is properly stored but let's see the moment in which Rails checks whether or not we have an entry for a certain item on our cache. Where this happens is in [ActionView::Helpers::CacheHelper](https://github.com/rails/rails/blob/914caca2d31bd753f47f9168f2a375921d9e91cc/actionview/lib/action_view/helpers/cache_helper.rb#L238)

```ruby
# rails/actionview/lib/action_view/helpers/cache_helper.rb

def fragment_for(name = {}, options = nil, &block)
    if content = read_fragment_for(name, options)
        @view_renderer.cache_hits[@current_template&.virtual_path] = :hit if defined?(@view_renderer)
        content
    else
        @view_renderer.cache_hits[@current_template&.virtual_path] = :miss if defined?(@view_renderer)
        write_fragment_for(name, options, &block)
    end
end

def read_fragment_for(name, options)
    controller.read_fragment(name, options)
end
```

We can see now how the _cache_ method in our view is just a helper that eventually calls the related controller to check whether the hashed key is available or not. So the actual link between our request and our cache will happen through the controller. I won't dive into it right now in order to stay on topic.
Let's use the same technique as before to cache a fragment of generated HTML by wrapping our `render post` with a call to `cache` on our posts#index view:

````ruby
# app/views/posts/index.html.erb
<% cache post do %>
    <%= render post %>
<% end %>
```ruby
````

Let's add an extra delay on the post view for demonstration purposes:

```ruby
# app/views/posts/_post.html.erb
<% sleep 1 %>
```

I added one entry to my posts on our website. Then if we have a look at the logs, we can see it takes a bit too long to render:

```bash
23:09:27 web.1  | Read fragment views/posts/index:9192fedf3654d46c8795f01c6eab374e/posts/6-20241126220920008737 (0.8ms)
23:09:28 web.1  |   Rendered posts/_post.html.erb (Duration: 1000.2ms | GC: 0.0ms)
```

but now SolidCache has saved our fragment and if we refresh, we'll see it's much faster:

```ruby
23:10:58 web.1  | Read fragment views/posts/index:9192fedf3654d46c8795f01c6eab374e/posts/6-20241126220920008737 (2.1ms)
23:10:58 web.1  |   Rendered posts/index.html.erb within layouts/application (Duration: 3.8ms | GC: 1.2ms)
```

Notice that the digest of this post is '6-20241126220920008737', as long as the record in our database is not update, that will always be retrieved from our db. As soon as this entry changes, a new digest will be created and the cache will be a miss next time we visit the site.
I updated the record on the website and when I enter the index page and see the logs, we'll see a new hash digest is used: 6-20241126221359025448

```ruby
23:14:04 web.1  | Read fragment views/posts/index:9192fedf3654d46c8795f01c6eab374e/posts/6-20241126221359025448 (0.9ms)
23:14:05 web.1  |   Rendered posts/_post.html.erb (Duration: 1000.2ms | GC: 0.0ms)
```
