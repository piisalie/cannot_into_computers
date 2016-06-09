---
title: LessCode Form Validation - Serverside Edition
date: 2016-06-09 22:51 UTC
tags: ruby
---

One common task that comes supported in most large frameworks is form validation.

There are *at least* three different places form validation could be implemented.

1. Client-side
2. Server-side
3. In the Database

For this post we'll mostly explore the Server-side variety, with an example of
how we can use the server to respond with helpful messages for the user.

I tend to think of Server-side validations as 'does it exist?' and I leave the
'is it correct?' validation to the database (or a consumer in the case of third
party services).

Libraries used: [Cuba](https://github.com/soveran/cuba), [Mote](https://github.com/soveran/mote)

Suppose we're building a simple meal planning app that allows a user to input
details about a recipe. A recipe has a 'name', a 'source' web address, and some
ingredients. We would like to require the inclusion of 'name' and 'ingredients'.
Whereas, the 'source' will be optional.

Cuba provides an 'on param' block that can be nested within a POST / route
block that is can be used to choose code paths based on which params exist.

Below is a possible POST /create route.

```ruby
  on post do
    on 'create' do
      on param('name'), param('source'), param('ingredients') do |name, source, ingredients|
        # All three params are accounted for
      end

      on param('name'), param('ingredients') do |name, ingredients|
        # The two required params are accounted for
      end

      on param('name') do |name|
        # missing ingredients
      end

      on param('ingredients') do |ingredients|
        # missing name
      end

      on true do
        # catch all
      end

    end
```

If you squint - this sort of looks like pattern matching based on POST params.

The first two blocks give us a place to handle the case where the two required
params are provided. We can pass them and the `res` response object Cuba
provides on to some sort of handler object.

In the next two blocks we handle cases where at least one required param is missing.
This gives us an entry point to re-render the new template with an error and
the form information that has already been submitted.

The last block is a catch-all.

Once we've matched a param, we can make use of it within the block. Below, we re-use
the submitted 'name' param to re-render the new view along with a helpful error message.

```ruby
      on param('name') do |name|
        error_message = "Cannot save a recipe without Ingredients"
        res.write(view('recipes/new', error: error_message, name: name))
      end
```

We can present the error in the `new` partial, and pass along the already
given name.

```html
<!-- ... -->

% if error
  <p>{{ error }}</p>
% end

<p>
  <form action="/recipes/create" method="post">
    <label>Name</label>
    <input type="text" name="name", value="{{name}}"/>

<!-- .... -->
```

Now we *always* have to send along the `error` and the `name` variables when
rendering the view. (including in the case of the `new` route)

```ruby
  on 'new' do
    res.write(view('recipes/new', name: "Enter name", error: nil))
  end
```

While I like that we're building up a representation of user state, it is kind
of annoying to include all the `nil` params when setting the initial state. One
way to solve this is by adding a wrapper/helper method for the `res.write`
interface to set defaults for you.

```ruby
  def render_new(response:, name: nil, error: nil, ingredients: nil)
    params = { name: name, error: error, ingredients: ingredients}
    response.write(view('recipes/new', params))
  end
```

Now the `new` block doesn't have to worry about the extra view variables...

```ruby
  on 'new' do
    render_new(response: res)
  end
```

... and the post param blocks can still respond with helpful messages:

```ruby
      on param('ingredients') do |ingredients|
        error_message = "Cannot save a recipe without a name"
        render_new(response: res, error: error_message, ingredients: ingredients)
      end
```

In a way, we're representing the user's state (including currently entered
values, and existing errors) as a hash of params to be presented in the view.
While this is a simple example, hopefully it provides some groundwork for easily
rolling your own form validations with less magic.
