---
title: Polymorphic Spree
date: 2016-04-14 20:19 UTC
tags: ruby
---
This post will try to describe polymorphism using a simple example and compare that solution with a semi-stateless message passing based approach. I was inspired to write this by [JEG2's observations](http://tech.noredink.com/post/142689001488/the-most-object-oriented-language) in using Elixir and how it relates to Object Oriented design, and also by Avdi Grim's wonderful [Ruby Tapas](http://www.rubytapas.com/) episode 357 'Object Oriented Programming'. If you're not already a Ruby Tapas subscriber I highly recommend it.

### Polymorphism

I did not come from a Computer Science background, and for a long time I did not have a good understanding of what polymorphism is. The [wiki](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)) has some examples and a lot of large words. But I understand things better when I can write them out so I'd like to begin with a simple example of polymorphism in Ruby.

Let's say we have a few shape classes, and we'd like to compute the area of each shape. When I was first learning Ruby I probably would have written something like:

```ruby
class Triangle
  attr_reader :base, :height
  def initialize(base:,height:)
    @base   = base
    @height = height
  end
end

class Square
  attr_reader :side_length
  def initialize(side_length:)
    @side_length = side_length
  end
end

def calculate(shape)
  if shape.class == Triangle
    (shape.base * shape.height) / 2
  elsif shape.class == Square
    shape.side_length ** shape.side_length
  else
    raise "I'm sorry Dave but I'm afraid I can't do that: #{shape.inspect}"
  end
end

square = Square.new(side_length: 2)
calculate(square)
# => 4
```

In spite of this being a contrived and simple example, we can see how every time a new shape is defined we would have to add it to the conditional. This is not ideal, and if you have worked in a large application you have likely come across nested and hard to follow `if .. else` blocks. So as Object Oriented developers we try to avoid this [anti-pattern](http://c2.com/cgi/wiki?ArrowAntiPattern) with polymorphism.

We can implement it simply by pushing the logic required to calculate the area of a shape into the shape itself.

```ruby
class Triangle
  attr_reader :base, :height

  def initialize(base:,height:)
    @base   = base
    @height = height
  end

  def area
    (base * height) / 2
  end
end

class Square
  attr_reader :side_length
  def initialize(side_length:)
    @side_length = side_length
  end

  def area
    side_length ** 2
  end
end

def calculate(shape)
  shape.area
end

square = Square.new(side_length: 2)
calculate(square)
# => 4
```

Now the `calculate` method has no logic, it relies an `area` method being defined on the shapes we pass in. We say objects that satisfy this constraint have the same interface and we have taken care of our ungainly conditional by using polymorphism. The logic for each shape's area computation is nicely encapsulated within the object.

### The Mutable State

One thing to consider about this approach is state. The `Square` is built before it's passed into the `calculate` method and while we don't currently have a way to change the properties of a shape after it's created, it is easy to see how its internal state could get altered (especially in parallel environments). A change could happen before the calculate method has a chance to run and possibly lead to unexpected results. We can work around this is by using a sort of message passing. By passing all of the object's intended state into the method we can enforce exactly what state we're calculating on while still keeping a top level `calculate` method pretty general.

```ruby
class Triangle
  attr_reader :base, :height
  def self.area(attributes)
    new(attributes).area
  end

  def initialize(base:,height:)
    @base   = base
    @height = height
  end

  def area
    (base * height) / 2
  end
end

class Square
  attr_reader :side_length
  def self.area(attributes)
    new(attributes).area
  end

  def initialize(side_length:)
    @side_length = side_length
  end

  def area
    side_length ** 2
  end
end

def calculate(shape, type, attributes)
  shape.send(type, attributes)
end

calculate(Triangle, :area, base: 2, height: 3)
# => 3

calculate(Square, :area, side_length: 2)
# => 4
```

The difference here is subtle, the shape is now created as a part of the calculation instead of the shape existing apart from it. The calculate method is a common entry point for any shape to return its area and we pass it messages containing the type of shape, which calculation, and the attributes for the shape. Later this could be expanded to work for messages containing calculations for things in addition to `area`. It is generic, extendable, and we have avoided both conditionals and state.


### But what does this have to do with anything?

In [JEG2's post](http://tech.noredink.com/post/142689001488/the-most-object-oriented-language) he explains how Elixir's processes and message passing enforce a sort of process based paradigm of Object Oriented design which is a bit different from what we call polymorphism in the Ruby world. I found this to be a super interesting point, and I wanted to use a small example to try and tease out differences between a more isolated message passing design and a simple polymorphic one in Ruby (since it's what I know).

We took a polymophic approach and converted it into something that relies on particular messages being passed. The message includes not only what type of object and which calculation to complete, but it also includes information about the state the object should be in for the calculation. The strengths of such an approach would be even more pronounced in a language such as Elixir which has robust pattern matching and where everything is just a process where passing messages is mandatory. This thought experiment certainly leaves me wanting to start digging into a new language. :)
