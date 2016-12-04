---
layout: post
title: C'mon Ruby People
date: 2016-11-08
tag: markdown
blog: true
#star: true
---

The Ruby community is one of the best I've ever had the chance to participate
in. We've got strong values on code quality, tests and best practices but
sometimes it feels like a religion.

By now, most of you have heard [DHH](https://twitter.com/dhh) preaching
about this `N+1 being a feature`. If not, just read
[this](https://rossta.net/blog/n+1-is-a-rails-feature.html) and come back
after.

The gist of the issue is:

    When a page (a blog index for instance) starts to slow down, the first thing
    to look for is N+1 queries. We solve that using :include to include every
    relation that is referenced on our views.

    If after all of our N+1 queries are solved our app is still slower than
    what we would like, we then reach out for view caching. But our views are
    cached in our .html.erb files, meaning our :include statement will still
    run and if we're including lots of relations this will still slow down our
    request even if we hit the cache.

    DHH basically says, remove the :include statement and let your uncached
    requests hit the N+1 queries.

To me this sounds a little worrying. I mean, we all have done **hacks** like this to
save our asses in production and **that's ok**. Software development is all about
the trade-off of what you can build in the time frame that it actually is
useful.

The problem with this trade-off in particular is that the worst
case-scenario (stale cache) might be extremely slow.

On one case at [Nourish](http://nourishcare.co.uk), the difference between a
request with and without including the referenced models would drive the
response time from around 600 ms to 45 seconds.

Let's put that in perspective:

- No cache, referenced models NOT included: 45 s (god, that's slow)
- No cache, including referenced models: 600 ms (ok'ish, but slow)
- Cache, including referenced models: 500 ms (not much faster)
- Cache, not including referenced models: 80 ms (best case) / 45 seconds (worst
  case)

This can't be something that we preach as a feature or the right way to do
caching. And what worries me the most is that as a community we've mostly
embraced this sort of hacks under the umbrella of the "Rails way" without
asking if it really is something we should be doing.

Ok, enough ranting. Let's take a look at what might be wrong and a possible
solution to fix it without the original workaround drawbacks.

# Smell #1 - Why U No Faster?

The first thing that starts to stink with this approach is the fact that the
cache layer is not helping at all. This happens because for this particular
request the most costly operation is instantiating all ActiveRecord objects and
their associations, **not** rendering the view.

Caching is about saving the results of an expensive computation that is pure
(given the same input, it will always produce the same output with no
side-effects).

If the cached version is not faster than the initial version, then we're
probably caching at the wrong level or the computation is not expensive at all.

# Smell #2 - But you said I should include all the things

Removing the included associations might mean a substantial slow down of the
request for the case where we're not hitting the cache (remember 600 ms is
faster than 45 s).

If your application has frequent writes, hitting a stale cache might not be
rare at all and be frustrating a big number of your users.

# Smell #3 - Cache is logic

Caching mechanisms are part of your application logic, they are conditionals.
The same way you don't want logic in your views, you probably don't want cache
either.

This doesn't mean you should never have cache in your views though. Be
pragmatic. The same way an if statement on your view won't render you
application a nightmare to maintain, a `cache block` won't either.

Just keep in mind that there are other alternatives, specially when what you
want to cache might not be the view at all but the code that is producing the
input for your view layer (an expensive query for instance).

# Ok. How do we solve it then?

I know I started this with a rant, but it might just be the case that for you,
introducing cache and removing preloaded associations is good enough. If your worst
case scenario is fast enough, than there's no good reason to introduce extra
complexity (but in that case, why did you need cache anyway?).

When in doubt, measure.

## Putting the cache in its place

Since in this case the most expensive thing we're doing is query'ing and
instantiating records, we need to move the cache to the point where this operation
is being computed. Which in this case would be the controller.

The problem with that approach is that we would be stuck with caching a single
level (no russian doll cache for you).

So what I like to do instead, is to create small components for this. **Bare
with me**, I'm **not** going all **React** on you.

These components will preload associations if needed and contain your view logic
as well, essentially keeping all logic out of your views. You get to keep your
view logic tidy, nestable components and a great place to cache expensive work
as well.

## Hmmm. Components?

Let's imagine a simple blog index page and build a component for the posts list.

A component is simply a small PORO that will render a given view:

```ruby
class BlogPostsComponent
  attr_reader :blog_posts

  def initialize(blog_posts)
    @blog_posts = blog_posts
  end

  def render
    @blog_posts = blog_posts.includes(:author)
    render_to_string 'components/blog_posts.html.erb', locals: { component: self }
  end
end
```

As you can see, there's nothing scary or complex here. We're just bringing the
concept of a partial to our code.

Our partial view will stay exactly the same:

```html
<% component.blog_posts.each do |post| %>
  <h1><%= post.title %></h1>
  <div><%= post.body %></div>
  <div><%= post.author %></div>
<% end %>
```

Your controller won't change dramatically:

```ruby
class BlogPostsController < ApplicationController
  # BEFORE - The Rails way
  def index
    @posts = BlogPost.all.includes(:author)
  end

  # AFTER - With components
  def index
    posts = BlogPost.all
    @posts_component = BlogPostsComponent.new(posts)
  end
end
```

And your view will still be much like rendering a partial:

```html
<!-- BEFORE - The Rails way -->
<h1>All Posts</h1>
<%= render @posts %>

<!-- AFTER - With components -->
<h1>All Posts</h1>
<%= @posts_component.render %>
```

With components you get a place for you logic view that you can easily test and
maintain. Your views will be as dumb as possible and as you will see in a moment
you'll get a place where you can use your cache without introducing N+1
problems.

## Caching done right

Now that we've got our component set up, caching our posts is as simples as:

```ruby
class BlogPostsComponent
  def render
    Rails.cache.fetch(@blog_posts) do
      @blog_posts = blog_posts.includes(:author)
      render_to_string 'components/blog_posts.html.erb', locals: { component: self }
    end
  end
end
```

As you can see, we're including the associations inside the cache block. This
means that if we hit the cache no association preloading will take place, but
if we miss it we will still preload the authors.

So basically, you keep the cake and eat it too.

## Further improvements

Now that we've got our components working, we can clean this up a bit by
introducing an `ApplicationComponent`:

```ruby
class ApplicationComponent
  include ActionView::Rendering

  protected

  def cache(key, &block)
    Rails.cache.fetch(key, &block)
  end

  def render_template(template)
    render_to_string(template, locals: { component: self })
  end
end
```

So that our `BlogPostsComponent` becomes:

```ruby
class BlogPostsComponent < ApplicationComponent
  attr_reader :blog_posts

  def initialize(blog_posts)
    @blog_posts = blog_posts
  end

  def render
    cache(blog_posts) do
      @blog_posts = blog_posts.includes(:author)
      render_template 'components/blog_posts.html.erb'
    end
  end
end
```

If our individual blog entries start to grow in complexity we might want to
create a component for them as well:

```ruby
class BlogPostComponent < ApplicationComponent
  attr_reader :blog_post

  LENGTH_CAP = 255

  def initialize(blog_post)
    @blog_post = blog_post
  end

  def title
    blog_post.title.titleize
  end

  def body
    return blog_post.body if blog_post.body.length < LENGTH_CAP
    "#{blog_post.body[0..max_length - 3]}..."
  end

  def author_name
    blog_post.author.name
  end

  def render
    render_template 'components/blog_post.html.erb'
  end
end
```

This way we're able to get rid of all the logic in our views, which will improve
both maintainability and testability.

# Summing it up

The Rails Way is great and provides great agility in the beginning of a project,
but as our code base grows in size and complexity we need a different set of
abstractions to cope with that.

In the case of views, components are a great way to structure your code from the
controller to the view level:

  - Components are a good pattern if your views are growing larger and more complex.
  They are the best place to cache parts of your pages and to put your logic view.

  - Preloading associations is usually tightly coupled with your views, so
  moving them from the controller to the component (basically a presenter that
  knows how to render its view), means you're grouping two bits of logic that
  usually change at the same time - this is a good indicator that your classes are
  following the SRP.

  - Cache in your views is **NOT** the root of all evil. But keep an eye open for
  smells such as degraded performance for worst case scenarios.
