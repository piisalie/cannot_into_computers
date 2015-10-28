---
title: On Selling Used Cars With Postgres
date: 2015-10-22 23:34 UTC
tags: ruby, postgres
---

## On Selling Used Cars with Postgres

Let's say we have an inventory of used vehicles and we'd like to
display a selection of these vehicles to a user. It would be
ideal if we showed the user a group varied by brand / model.
Toyota makes a great car but It wouldn't do to only display
Toyota Camrys over and over.

It's not difficult to imagine a car as a simple data object:

```ruby
cars = [
  { make: "Toyota", model: "Camry", year: 1996, mileage: 201293, color: "Green" },
  { make: "Toyota", model: "Camry", year: 2011, mileage: 11920, color: "Black" },
  { make: "Toyota", model: "Highlander", year: 2009, mileage: 47560, color: "Silver" },
  { make: "Chevrolet", model: "HHR", year: 2007, mileage: 128743, color: "Sandstone Metallic" }
]
```

Ideally if we ask to display two cars we'd get a Toyota
and a Chevrolet. If we ask for three cars we'd get a two different
Toyotas (a Camry and a Highlander) and a Chevrolet.

It isn't too hard to get these two tests passing:

```ruby
def sample(count, list)
  makes = list.map { |vehicle| vehicle[:make]}.uniq.cycle

  list.select { |vehicle| vehicle[:make] == makes.next }.take(count)
end

describe 'sample' do

  it 'returns a good distribution of makes' do
    sample = sample(3, cars)
    makes  = sample.map { |vehicle| vehicle[:make] }
    expect(makes.sort).to eq(["Toyota", "Toyota", "Chevrolet"].sort)
  end

  it 'returns a good distributions of models' do
    sample  = sample(3, cars)
    models  = sample.map { |vehicle| vehicle[:model] }
    expect(models.sort).to eq(["Camry", "Highlander", "HHR"].sort)
  end

end
```

The bad news is they're false positives. If we change the order of the cars
the test for distribution of models fails.

We can modify the test to fail sometimes by shuffling the cars array on each
test run, but I don't particularly like that idea. There's no kill like over
kill, so we can test all possible permutations of the `cars` array:

```ruby
  it 'returns a good distributions of models' do
    cars.permutation.to_a.each do |permutation|
      sample  = sample(3, permutation)
      models  = sample.map { |vehicle| vehicle[:model] }
      expect(models.sort).to eq(["Camry", "Highlander", "HHR"].sort)
    end
  end
```

This fails, but not in the way I expect. Interestingly, there is a particular
order of `cars` where ALL the tests fail. I find this hilarious and points out a
glaring flaw in my implementation.

If the ordering of `cars` by make is `[ 'Toyota', 'Toyota', 'Chevrolet', 'Toyota' ]`
and the ordering of the `makes` enumerator is `[ 'Toyota', 'Chevrolet', 'Toyota', 'Chevrolet' ]`
it is possible to return only ONE sample. We got lucky with select.

Maybe we should step back: what do we actually want to do?
We want to take a car with a matching make from the `makes` enumerator
(preferably one we haven't seen before). This fixes the first test and gives
an expected failure on the last test:


```ruby
def sample(count, list)
  makes = list.map { |i| i[:make]}.uniq.cycle
  cars_by_make = list.group_by { |car| car[:make] }

  samples = [ ]

  count.times do
    possibility = cars_by_make[makes.next].shift
    redo unless possibility
    samples << possibility
  end

  samples
end
```

The test for Makes distribution is a bit more tricky. Initially I had thought
we needed a product of the list of unique makes and unique models that we
could use to choose values from cars:

```ruby
def sample(count, list)
  makes  = list.map { |i| i[:make]}.uniq
  models = list.map { |i| i[:model]}.uniq

  pairings = makes.product(models).cycle

  samples = [ ]

  count.times do
    make, model = pairings.next
    possibility = list.find { |car| car[:make] == make && car[:model] == model }
    redo unless possibility
    samples << possibility
  end

  samples
end
```

This passes the tests, but if you're paying attention you may have
noticed that we generate ALL possible combinations of Make and Model.
At best this is skipped over, since the pairings do not exist in the
data. At worst it upsets some auto manufacturer's legal departments with
things like "Toyota HHR"s.

If you're REALLY paying attention you may have noticed what we're
actually doing is re-implementing a self `CROSS JOIN`. We can net
similar results with a relational database.

EDIT: The following only accidentally correct and somewhat silly. I think
what we actually want here is `WITH RECURSIVE` but it probably warrants
a post of its own.

```
cross_joining=# SELECT * FROM vehicles;                                                                                                                                                           make    |   model    | year | mileage |       color        
-----------+------------+------+---------+--------------------
 Toyota    | Camry      | 1996 |  201293 | Green
 Toyota    | Camry      | 2011 |   11920 | Black
 Chevrolet | HHR        | 2007 |  128743 | Sandstone Metallic
 Toyota    | Highlander | 2009 |   47560 | Silver


cross_joining=# SELECT DISTINCT make, model FROM vehicles WHERE
  (vehicles.make, vehicles.model) IN
     (SELECT make, model FROM
       (SELECT DISTINCT vehicles.make FROM vehicles) as j1
       CROSS JOIN
       (SELECT DISTINCT vehicles.model FROM vehicles) as j2);

   make    |   model
-----------+------------
 Toyota    | Camry
 Chevrolet | HHR
 Toyota    | Highlander
(3 rows)

```

We're selecting the same varied group of makes and models, as in the Ruby example.
Things tend to get a little odd as the table size grows. The query is quite performant,
(even on tables over 100_000) but does not always give the results I'd prefer / expect.

Another weakness of this approach is we always get the same ordering of results. So if
we wanted to show the user a different selection of vehicles on the next visit it can't
quite deliver. Perhaps in a later exercise I'll try and find solutions to these problems.

Code from this post can be found [here](https://gist.github.com/piisalie/c913bdbfcf9211d9f927)

Changes for this article can be found [here](https://github.com/piisalie/cannot_into_computers/commits/master/source/articles/2015-10-22-on-selling-used-cars-with-postgres.html.markdown)

Notes:

* This only SORT OF works, and need further improvements.
* This is probably totally wrongheaded and my SQL is terrible.
* I welcome corrections in the form of issues or PRs on this blog's [repo](https://github.com/piisalie/cannot_into_computers)
* I actually drive a Toyota Camry with ~200_000miles.


Special thanks to [Nathan Long](https://twitter.com/sleeplessgeek) for proof
reading, feedback, corrections, and being a generally awesome person.
