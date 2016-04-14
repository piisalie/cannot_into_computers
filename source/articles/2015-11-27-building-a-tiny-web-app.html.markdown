---
title: Building A Tiny Web App
date: 2015-11-27 18:30 UTC
tags: ruby, lesscode
---
I've been infatuated with the LessCode movement as of late. A few days ago I set out
to see how quickly I could construct a small database backed application. Granted
it did take a bit longer than 15 minutes, the libraries used in this project are
quite small and pretty easy to understand. We'll avoid magic, and walk through each
step.

Libraries used: [Cuba](https://github.com/soveran/cuba),
[Mote](https://github.com/soveran/mote), [Nomadize](https://github.com/piisalie/nomadize),
[SQLCapsule](https://github.com/piisalie/sql_capsule)

We'll be building a super simple used car inventory app. Sometimes I like to start
with the web interface when building an application, so we'll setup Cuba and make
a few small routes for viewing and adding vehicles to the inventory.

    $ mkdir inventory_app && cd inventory_app
    $ touch Rakefile Gemfile

Setup the Gemfile:

```ruby
# Gemfile
source 'https://rubygems.org'

gem 'rake'
gem 'cuba'
gem 'mote'
gem 'mote-render'
```

Now we can bundle install and get started on defining the routes, we're going to
need to build some views so we'll go ahead and create those directories and views.

    $ bundle install
    $ touch app.rb
    $ touch config.ru
    $ mkdir -p app/views
    $ touch app/home.mote app/layout.mote

```ruby
# app.rb
require 'cuba'
require 'mote'
require 'mote/render'

# setup mote renderer
Cuba.plugin(Mote::Render)

# tell mote where to find its view files
Cuba.settings[:mote][:views] = "./app/views/"

Cuba.define do

  on root do
    # render app/views/home.mote
    res.write(view("home"))
  end

end
```

config.ru is pretty basic it just launches the app:

```ruby
# config.ru
require './app'

run(Cuba)
```

layout.mote is also pretty simple:

```html
<html>
  <head>
     <title>Inventory App</title>
  </head>
  <body>
    {{ content }}
  </body>
</html>
```

and lastly home is just a simple `H1` and a link to create a vehicle.

```html
<h1>Car Inventory</h1>
<a href="vehicles/new">Add Vehicle</a>
```

At this point you should be able to use `$ rackup` and actually visit `localhost:9292`
and see your home template rendered.

The add vehicle route and view don't exist yet, so we'll add them:

```ruby
  on "vehicles/new" do
    res.write(view("vehicles/new"))
  end
```

The corresponding view:

    $ mkdir -b app/views/vehicles
    $ touch app/views/vehicles/new.mote

A vehicle should have a 'make', 'model', 'mileage', and 'color', so we build
a simple form for that:

```html
<form action="/vehicles/create" method="post">
  <input type="text" name="make" value="make">
  <input type="text" name="model" value="model">
  <input type="text" name="mileage" value="mileage">
  <input type="text" name="color" value="color">

  <input type="submit" value="Submit">
</form>
```

the `vehicles/create` route 404's when we click the 'submit' button. We should
add a post route to our app.rb file:

```ruby
# ...
  on post do

    on "vehicles/create" do
        on param("make"), param("model"), param("mileage"), param("color") do |make, model, mileage, color|
          puts "make: #{make}, model: #{model}, mileage: #{mileage}, color: #{color}"
          res.redirect "/"
      end
    end

  end
# ...
```

Now when we fill in some details and click 'submit' we get our expected input
in the log.

```
make: Toyota, model: Camry, mileage: 201593, color: Green
127.0.0.1 - - [26/Nov/2015:10:07:04 -0600] "POST /vehicles/create HTTP/1.1" 302 - 0.0018
127.0.0.1 - - [26/Nov/2015:10:07:04 -0600] "GET / HTTP/1.1" 200 152 0.0007
```

Outputting to the log is cool, but not very useful, we'd really like to persist this so
we can view it later.

To do this we'll have to setup postgres using the Nomadize gem. We'll update the
Gemfile to include Nomadize:

```ruby
# Gemfile
gem nomadize
```

then

    $ bundle install

and lastly we'll update our Rakefile to get the Nomadize rake tasks:

```ruby
# Rakefile
require 'nomadize/tasks'
```

Now we can use the migration/database utility tasks provided by Nomadize:

    $ bundle exec rake -T
    rake db:create                         # Create database using name in {appdir}/config/database.yml
    rake db:drop                           # drop database using name in {appdir}/config/database.yml
    rake db:generate_template_config       # generate template config file
    rake db:migrate                        # Run migrations
    rake db:new_migration[migration_name]  # Generate a migration template file
    rake db:rollback[steps]                # rollback migrations (default count: 1)
    rake db:status                         # view the status of known migrations

We'll generate the template config and update it with our details:

    $ bundle exec rake db:generate_template_config
    Config created in config/database.yml

Now if we open that file we see basic database config file, let's update it to
include the database names:

```yml
# ./config/database.yml
---
development:
  :dbname: 'inventory_app_development'
test:
  :dbname: 'inventory_app_test'
production:
  :dbname: 'inventory_app_production'
```

Now we can go back to the console and use the rake task to create the database:

    $ bundle exec rake db:create
    CREATE DATABASE inventory_app_development;

Great, so we have a database, but we still need migrations:

    $ bundle exec rake db:new_migration[add_vehicles_table]

This creates a file in `./db/migrations` for us to add our migrations:

```yml
# ./db/migrations/[timestamp]_add_vehicles_table.yml
---
:up: 'CREATE TABLE vehicles (
        make    VARCHAR(50) NOT NULL,
        model   VARCHAR(50) NOT NULL,
        mileage INTEGER     NOT NULL,
        color   VARCHAR(50) NOT NULL,
        id      SERIAL
        );'
:down: 'DROP TABLE vehicles;'
```

Run the migration, and check the status:

    $ bundle exec rake db:migrate
    $ bundle exec rake db:status
    filename                          |  status
    -------------------------------------------
    20151126162330_add_vehicles_table |      up

So we've got a table, now let's put things in it. To do that we'll use SQLCapsule.
Add it to the Gemfile and Bundle.

```ruby
# Gemfile
# ...
gem 'sql_capsule'
```

    $ bundle install

To hook it together we require `nomadize/config` and `sql_capsule` and then use them
to actually insert a vehicle into our database:

```ruby
# app.rb
require 'cuba'
require 'mote'
require 'mote/render'
require 'nomadize/config'
require 'sql_capsule'

Cuba.plugin(Mote::Render)
Cuba.settings[:mote][:views] = "./app/views/"

Cuba.define do

  on root do
    res.write(view("home"))
  end

  on "vehicles/new" do
    res.write(view("vehicles/new"))
  end

  on post do
    on "vehicles/create" do
      on param("make"), param("model"), param("mileage"), param("color") do |make, model, mileage, color|
        db              = Nomadize::Config.db
        vehicle_queries = SQLCapsule.wrap(@db)
        save_query = "INSERT INTO vehicles (make, model, mileage, color) VALUES ($1, $2, $3, $4) RETURNING id;"
        vehicle_queries.register :add_vehicle, save_query, :make, :model, :mileage, :color

        result = vehicle_queries.run :add_vehicle, make: make, model: model, mileage: mileage, color: color
        puts result
        res.redirect "/"
      end
    end
  end

end
```

It seems like we can add vehicles, but it'd be better if we could list the vehicles
on the home page to be sure:

```html
<!-- home.mote -->
<h1>Car Inventory</h1>
<a href="vehicles/new">Add Vehicle</a>
<ul>
% vehicles.each do |vehicle|
  <li>{{ vehicle }}</li>
% end
</ul>
```

And we'll need to update the `on root` handler to pass in the list of vehicles:

```ruby
# app.rb
# ...
  on root do
    db               = Nomadize::Config.db
    vehicle_queries  = SQLCapsule.wrap(db)

    all_vehicles_sql = "SELECT * FROM vehicles;"
    vehicle_queries.register :all_vehicles, all_vehicles_sql

    vehicles = vehicle_queries.run(:all_vehicles)
    res.write(view("home", vehicles: vehicles))
  end
```

The `root` handler now shares a lot of database setup with our `vehicles/create` handler,
so we'll push it out into a `DB` class that is responsible for holding our registered
queries.

```ruby
# app/models/db.rb
require 'nomadize/config'
require 'sql_capsule'

class DB
  def self.vehicle_queries
    @vehicle_queries ||= begin
      query_holder = SQLCapsule.wrap(Nomadize::Config.db)
      setup_vehicle_queries(query_holder)
    end
  end

  private

  def self.setup_vehicle_queries(holder)
    all_vehicles_sql = "SELECT * FROM vehicles;"
    holder.register :all_vehicles, all_vehicles_sql

    save_vehicle_sql = "INSERT INTO vehicles (make, model, mileage, color) VALUES ($1, $2, $3, $4) RETURNING id;"
    holder.register :add_vehicle, save_vehicle_sql, :make, :model, :mileage, :color

    holder
  end
end
```

Now we can access saved vehicle queries by using `DB.vehicle_queries` and our app ends up
looking something like this:

```ruby
# app.rb
require 'cuba'
require 'mote'
require 'mote/render'

require_relative 'app/models/db'

Cuba.plugin(Mote::Render)
Cuba.settings[:mote][:views] = "./app/views/"

Cuba.define do

  on root do
    # get the vehicles from the database
    vehicles = DB.vehicle_queries.run :all_vehicles

    # pass the vehicles into the template
    res.write(view("home", vehicles: vehicles))
  end

  on "vehicles/new" do
    res.write(view("vehicles/new"))
  end

  on post do
    on "vehicles/create" do
      on param("make"), param("model"), param("mileage"), param("color") do |make, model, mileage, color|
        # save the vehicle
        id = DB.vehicle_queries.run :add_vehicle, make: make, model: model, mileage: mileage, color: color

        # log the id and redirect to root
        puts id
        res.redirect "/"
      end
    end
  end

end
```

The last thing we need is a way to mark a vehicle as sold. To do this we should
add a sell button to the root page along with a field to enter the sold amount.

```html
<!-- home.mote -->
<h1>Car Inventory</h1>
<a href="vehicles/new">Add Vehicle</a>
<ul>
% vehicles.each do |vehicle|
  <li>
    {{ vehicle }}
    <form action="/sales/create" method="post">
      <input type="hidden" name="vehicle_id" value={{ vehicle["id"] }}>
      <input type="text"  name="amount" value="amount">
      <input type="submit" value="Sold!">
    </form>
  </li>
% end
</ul>
```

We update the app file to support the route and output the params.

```ruby
# app.rb
# ...
    on "sales/create" do
      on param("vehicle_id"), param("amount") do |vehicle_id, amount|
        puts "vehicle_id: #{vehicle_id}, amount: #{amount}"
        res.redirect "/"
      end
    end
# ...
```

And see that they come through:

```
127.0.0.1 - - [27/Nov/2015:11:29:44 -0600] "GET / HTTP/1.1" 200 2081 0.0050
vehicle_id: 3, amount: 2500
127.0.0.1 - - [27/Nov/2015:11:29:48 -0600] "POST /sales/create HTTP/1.1" 302 - 0.0023
127.0.0.1 - - [27/Nov/2015:11:29:48 -0600] "GET / HTTP/1.1" 200 2081 0.0007
```

Now we need a sales table:

    $ bundle exec rake db:add_migration[add_sales_table]

And fill in the details:

```yml
# ./db/migrations/[timestamp]_add_sales_table.yml
---
:up: 'CREATE TABLE sales (
       id         SERIAL,
       amount     INTEGER NOT NULL,
       vehicle_id INTEGER NOT NULL
       );'
:down: 'DROP TABLE sales;'
```

And migrate:

    $ bundle exec rake db:migrate

We'll update our `DB` class, and add another class level method/instance variable
for sales queries:

```ruby
# /app/models/db.rb
# ...
  def self.sales_queries
    @sales_queries ||= begin
      query_holder = SQLCapsule.wrap(Nomadize::Config.db)
      setup_sales_queries(query_holder)
    end
  end
# ...
  private

  def self.setup_sales_queries(holder)
    save_sale_sql = "INSERT INTO sales (vehicle_id, amount) VALUES ($1, $2) RETURNING id;"
    holder.register :add_sale, save_sale_sql, :vehicle_id, :amount

    holder
  end
# ...
```

Now we can hook it together:

```ruby
# app.rb
# ...
    on "sales/create" do
      on param("vehicle_id"), param("amount") do |vehicle_id, amount|
        id = DB.sales_queries.run :add_sale, vehicle_id: vehicle_id, amount: amount
        puts id
        res.redirect "/"
      end
    end
# ...
```

This creates sales, but our root index page still shows all the vehicles
(even the sold ones), that seems bad. Let's add another vehicle query that
gets just the available vehicles (that is the ones that aren't in the sales
table)

```ruby
# app/models/db.rb
# ...
def self.setup_vehicle_queries(holder)
    # ...
    available_vehicles_sql = "SELECT * FROM vehicles LEFT JOIN sales ON vehicles.id=sales.vehicle_id WHERE sales.vehicle_id IS NULL;"
    holder.register :available_vehicles, available_vehicles_sql

    holder
end
# ...
```

We update the `root` handler to use available_vehicles instead:

```ruby
# app.rb
# ...
  on root do
    vehicles = DB.vehicle_queries.run :available_vehicles
    res.write(view("home", vehicles: vehicles))
  end
# ...
```

Now when we restart the app we get... an ERROR OMG:

```
[2015-11-27 12:00:49] INFO  WEBrick::HTTPServer#start: pid=3051 port=9292
SQLCapsule::Wrapper::DuplicateColumnNamesError: Error duplicate column names in resulting table: ["make", "model", "mileage", "color", "id", "id", "amount", "vehicle_id"]
This usually happens when using a `JOIN` with a `SELECT *`
You may need use `AS` to name your columns.
QUERY: SELECT * FROM vehicles LEFT JOIN sales ON vehicles.id=sales.vehicle_id WHERE sales.vehicle_id IS NULL;
```

So it looks like our join is returning a bunch of columns we don't really care about:
`["make", "model", "mileage", "color", "id", "id", "amount", "vehicle_id"]` let's update
the query to just grab the vehicle information.

```ruby
# app/models/db.rb
# ...
  def self.setup_vehicle_queries(holder)
  # ...
    available_vehicles_sql = "SELECT vehicles.make, vehicles.model, vehicles.mileage, vehicles.color, vehicles.id FROM vehicles LEFT JOIN sales ON vehicles.id=sales.vehicle_id WHERE sales.vehicle_id IS NULL;"
    holder.register :available_vehicles, available_vehicles_sql

    holder
  end
# ...
```

Now our page loads again, and we see only the available vehicles. We can mark vehicles as
sold and they are removed from the list.

It would be great if now we could display the total sales, or a list of the sold vehicles
etc, but since this post is already eye glazingly long I'll leave that as an exercise
for the viewer. ;-)

Notes:
I'm not totally sold on my method of pushing the database setup into class methods
on the DB class. If we were building an actual app I think that would be difficult to
test and the `setup_vehicle_queries` method would likely get unruly over time. However,
for the purposes of this post/experiment I think it works alright.

Hopefully you now have a better idea of how easy it can be to build a database backed
web application using a minimal stack.

Our resulting project looks like:

```
.
├── app
│   ├── models
│   │   └── db.rb
│   └── views
│       ├── home.mote
│       ├── layout.mote
│       └── vehicles
│           └── new.mote
├── app.rb
├── config
│   └── database.yml
├── config.ru
├── db
│   └── migrations
│       ├── 20151126162330_add_vehicles_table.yml
│       └── 20151127173600_add_sales_table.yml
├── Gemfile
├── Gemfile.lock
└── Rakefile
```
