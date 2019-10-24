# README

There seems to be a bug in how Rails 6 collection cache versioning works. When
using a query with a LIMIT, the cache key might not change when a record is
destroyed.

- `collection.cache_key` remains unchanged because the query stays the same

- `collection.cache_version` remains unchanged too, because the record count stays
the same (due to LIMIT) and the `updated_at` of the most recently created record
stays the same too.

# Reproducing steps

- `bundle install`
- `yarn install`
- `rails db:migrate RAILS_ENV=development`
- `rails dev:cache`
- `rails s`
- open http://localhost:3000/posts
- create 6 posts (name them "post 1", "post 2", etc)
- of the 3 posts shown, destroy the middle one
- note how it's still included in the cached fragment

# Alternatively you could do in Rails console

```ruby
# Destroy all posts in case we have any
Post.destroy_all

# Create six records
6.times { |i| Post.create title: "Post #{i + 1}" }

# If we had an homepage, we might show the last 3 posts
latest_posts = Post.order(created_at: :desc).limit(3)

cache_version_1 = latest_posts.cache_version
# => 3-20191024183507613858
#
# 3 is the number of records
# 20191024183507613858 is the max updated_at timestamp

cache_key_1 = latest_posts.cache_key
# => posts/query-994c13f1de6fbfa86f588b6ff1b8cebf
#
# This is a hash of the query. Different queries generate different cache keys

# These are the titles we'd see on our homepage
titles_1 = latest_posts.pluck(:title)
# => ["Post 6", "Post 5", "Post 4"]

# Destroy the second post
latest_posts.second.destroy

# Refresh the homepage. Load 3 latest posts
latest_posts = Post.order(created_at: :desc).limit(3)

# Post 5 is gone, but Post 3 was added
titles_2 = latest_posts.pluck(:title)
# => ["Post 6", "Post 4", "Post 3"]

# So far everyting goes as expected. But imagine we used fragment caching on our
# homepage. Now we'd expect to show different posts, but you'll see below our
# cache keys remain unchanged. Meaning that our fragment cache would be
# outdated.

# These values will remain unchanged, which means Rails would show our outdated
# fragment cache
cache_version_2 = latest_posts.cache_version
cache_key_2 = latest_posts.cache_key

if cache_key_1 == cache_key_2 &&
   cache_version_1 == cache_version_2 &&
   titles_1 != titles_2

  p "Cache keys and versions are the same despite"
  p "the collection of records being different"
end
```
