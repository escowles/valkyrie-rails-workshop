# Valkyrie Rails

A giuded installation and build of a Rails application using the Valkyrie gem
as its database persistence layer.

The tutorial was written on MacOS, so some of the command examples will be
for that operating system, but most of the commands will be applicable on
any Unix-type OS.

## Requirements

* Ruby 2.6 or later
* Rails 5.2 or greater (6.0 is recommended)
* Git
* bundler 2.0.2 or greater
* Postgresql 9.2 or later (for jsonb support)

## Installing Requirements

    gem install rails bundler
    brew install postgresql

## Steps

### Part 0: Initialize Rails Application

Clone this repo to your system in a directory of your choice, then start off
with an empty Rails application using the provided template.

    cd valkyrie-rails
    rails new library-repo -m template.rb -T -d postgresql

### Part 1: Install and Configure Valkyrie

From this point forward, all your changes will be made in the newly created
`library-repo` application.

Add the valkyrie gem to your Gemfile and install.

    gem 'valkyrie'
    bundle install

Valkyrie needs to run some database migrations to prepare your application.
Install and run the provided migrations.

    bundle exec rake valkyrie_engine:install:migrations
    bundle exec rails db:create db:migrate

Lastly, we need to configure Valkyrie's storage and metadata persisters.

* copy [valkyrie.rb](examples/valkyrie.rb) to `config/initializers/valkyrie.rb`
* copy [valkyrie.yml](examples/valkyrie.yml) to `config/valkyrie.yml`

### Part 2: Create the Book Model and Test

Next, we'll create our first Valkyrie resource using a generator provided in the
gem. This will be the Book model with three basic fields.

    bundle exec rails g valkyrie:resource Book title:string author:string description:text

This should create the file `app/models/book.rb`. Note that while we have specified
different field types, Valkyrie creates attributes of all the same type,
a Valkyrie::Types::Set. By default, all attributes in Valkyrie resources are
arrays.

We can also create a spec test for our new model that uses shared specs from
the Valkyrie gem. Copy the included spec test to your spec folder.

    mkdir spec/models

Copy [book_spec.rb](examples/book_spec.rb) to the above directory. Then,
run the test suite.

    bundle exec rspec

You should see about 20 examples run. These are only the included specs from
the Valkyrie gem and are provided to ensure that your resources conform to the
current Valkyrie API.

### Part 3: Scaffolding Additional Code

We will need controllers, views, and other components in order to have a fully
functional application. Fortunately, we can use the generators provided by
Rails to create these additional components, and then modify them to work with
Valkyrie's persistence strategy.

To start, let's scaffold the additional pieces that are required.

    bundle exec rails g scaffold_controller Book title:string author:string description:text

This will create controllers and view code for our book. Note that we provide
the field information in order to generate input elements for our forms

Let's run the spec tests to see where we stand

    bundle exec rspec

That should show about 14 failures! No problem. The easiest place to start
is our routes, which were not created with the scaffolding command.

Add the following line to your `config/routes.rb` file:

    resources :books

Now, re-run your spec tests...

    bundle exec rspec

That allow all our routing tests to pass and knock us down to 6 failures.

### Part 4: Getting the Controller Working with Change Sets

The next place to look should be our controller, which is where our book
resources will be persisted to the database. The generated spec test is testing
each action in the controller, but it's assuming we're using a Rails' standard
ActiveRecord. Since most of the tests start off being skipped, we'll fix the
ones that are currently failing and then proceed to the others as needed.

The first test that's failing should be our `GET #index` action. For now, we
want our controller tests to render our views so that we can get more information
about what's going wrong. N.B. Typically, this isn't done and the views are
rendered in view tests, but we're going to take a shortcut.

Add the following line to your `books_controller_spec.rb` file, just under
the `RSpec.describe` block:

    render_views

Then run the spec test for the controller:

    bundle exec rspec spec/controllers/books_controller_spec.rb

Our `GET #new` should report a new error

    ActionView::Template::Error:
      undefined method 'errors' for #<Book:...

In a Rails application using ActiveRecord, models have an `errors` method
which returns a hash of errors that might have occurred during the create or
update process. Valkyrie::Resource objects have no such method because errors
are considered part of the form and not the model. Valkyrie does have form
objects called _change sets_ which have an `errors` method that can track
errors in a resource.

To fix this test, we can create a change set for our book and use that in the
controller instead of the resource itself.

First, let's make directories for our change sets and their tests.

    mkdir app/change_sets
    mkdir spec/change_sets

Change sets generally (but not always!) duplicate the exact same attributes
as the model and have additional validation specifications as well.
For now, we will create the simplest change set possible, and use shared specs
from the Valkyrie gem to test it.

* copy [book_change_set.rb](examples/book_change_set.rb) to `app/change_sets/book_change_set.rb`
* copy [book_change_set_spec.rb](examples/book_change_set_spec.rb) to `spec/change_sets/book_change_set_spec.rb`

Let's run the change set's spec tests to make sure everything is working as
expected:

    bundle exec rspec spec/change_sets

This should produce 18 passing examples.

Now that we have a working change set, all that we need to do is use it in
place of the resource in our `new` action. In the `books_controller.rb` file,
make the following change:

    def new
      @book = BookChangeSet.new(Book.new)
    end

Run-run your controller spec test...

    bundle exec rspec spec/controllers/books_controller_spec.rb

And now you should be down to only one failure!
