---
title: TDD - Top Down Design
date: 2015-09-21 12:30 UTC
tags: ruby
---
Note: This is post number one in a series on building an application without a proper ORM.

Backstory: For a long time, I've wanted an app I could put recipes in that showed each step
of the recipe, in order, on a singular page, with nothing else. I have trouble finding/keeping
my place when working through a recipe and this seemed like an easily solvable problem.

I intend to use/display the `Step` like (using [mote](https://github.com/soveran/mote) template engine):

```html
<step>
  % if step.has_ingredients?
  <ingredient>
    {{ step.ingredients }}
  </ingredients>
  % end
  <directions>
    {{ step.directions }}
  </direction>
</step>
```

Now I have a starting place for a few `Step` tests:

```ruby
describe Step do
  let(:directions)  { "Mix a lot, sir" }
  let(:ingredients) { [ :pears, :honey ] }

  def build_a_step(directions: "", ingredients: [])
    Step.new(directions: directions, ingredients: ingredients)
  end

  it "can have ingredients" do
    step = build_a_step(ingredients: ingredients)
    expect(step.ingredients).to eq(ingredients)
  end

  it "knows if it has any ingredients" do
    step = build_a_step
    expect(step.has_ingredients?).to be(false)

    step_with_ingredients = build_a_step(ingredients: ingredients)
    expect(step_with_ingredients.has_ingredients?).to be true
  end

  it "has directions" do
    step = build_a_step(directions: directions)
    expect(step.directions).to eq(directions)
  end
end
```

This is a pretty simple / easy to build class. Now we need a way of
finding/creating a `Step` for handing into the view. I don't want to think about
the database schema, so we'll just make a silly class method `find` that at
least pretends like it does what we need.

```ruby
it "has a .find interface" do
  dummy_db = double
  allow(dummy_db).to receive(:get_step)
    .with(recipe_id: 1, step_number: 2) { { directions: directions } }

  expect(Step.find(recipe_id: 1, step_number: 2, database_wrapper: dummy_db)).to be_a(Step)
end

class Step
  def self.find(recipe_id:, step_number:, database_wrapper:)
    new(database_wrapper.get_step(recipe_id: recipe_id, step_number: step_number))
  end
end
```

ERMEHGERD A MOCK!!! Is that terrible? Probably. But it affords us the ability to insert
a fake database wrapper with fake data and not even consider the database or schema.

```ruby
class FakeDB
  def self.get_step(recipe_id:, step_number:)
    recipes = [ [ { directions: "Mix a lot, sir!" } ] ]
    recipes[(recipe_id.to_i - 1)][(step_number.to_i - 1)]
  end
end
```

Note that I'm preemptively basing a `Step` on a recipe id which may or may not be a good idea,
time will tell. For now, it seems like we should have something that takes the recipe id,
and a step number and gives us a displayable `Step`.

For the application side: I've been tinkering with [cuba](https://github.com/soveran/cuba):

```ruby
on "recipes/(\\d+)-step-(\\d+)" do |recipe_id, step_number|
  step = Step.find(recipe_id: recipe_id, step_number: step_number, database_wrapper: FakeDB)
  res.write view("recipes/step", step: step)
end
```

This works for routes like `recipes/1-step-1`, and we have a place to put some fake recipe/step
data. We have less than 100 lines of code, and a pretty useless (but working) example of a
recipe step, but it's not much use without an actual recipe.

The commit from this post can be found [here](https://github.com/piisalie/recipeat/commit/df85555e95b6ee08d83ac020a61d3b3a2c6d5da5)

Coming soon: Post 2 - Something Something Recipes
