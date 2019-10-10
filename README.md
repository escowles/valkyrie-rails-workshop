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

    cd valkyrie-rails-workshop
    rails new library-repo -m template.rb -T -d postgresql

### Part 1: Install and Configure Valkyrie

From this point forward, all your changes will be made in the newly created
`library-repo` application.

    cd library-repo

Add the valkyrie gem to your Gemfile

    gem 'valkyrie'

Install the new gem by running

    bundle install

Valkyrie needs to run some database migrations to prepare your application.
Install and run the provided migrations.

    bundle exec rake valkyrie_engine:install:migrations
    bundle exec rails db:create db:migrate

Lastly, we need to configure Valkyrie's storage and metadata persisters.

    cp ../examples/valkyrie.rb config/initializers
    cp ../examples/valkyrie.yml config

### Part 2: Create the Book Model and Test

Next, we'll create our first Valkyrie resource using a generator provided in the
gem. This will be the Book model with three basic fields.

    bundle exec rails g valkyrie:resource Book title:string author:string description:text

This should create the file `app/models/book.rb`. Note that while we have specified
different field types, Valkyrie creates attributes of all the same type,
a Valkyrie::Types::Set. By default, all attributes in Valkyrie resources are
arrays.

We can also create a spec test for our new model that uses shared specs from
the Valkyrie gem. Copy the included spec test to your spec folder and
run the tests.

    mkdir spec/models
    cp ../examples/book_spec.rb spec/models
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

    cp ../examples/book_change_set.rb app/change_sets 
    cp ../examples/book_change_set_spec.rb spec/change_sets 

Let's run the change set's spec tests to make sure everything is working as
expected:

    bundle exec rspec spec/change_sets

This should produce 18 passing examples.

Now that we have a working change set, all that we need to do is use it in
place of the resource in our `new` action. In the `books_controller.rb` file,
make the following change:

``` ruby
def new
  @book = BookChangeSet.new(Book.new)
end
```

Run-run your controller spec test...

    bundle exec rspec spec/controllers/books_controller_spec.rb

And now you should be down to only one failure!

### Part 5: Creating Resources with the Controller

The next controller test to fix is the `create` action. Currently, you should
see an error like:

    NoMethodError: undefined method 'count' for Book:Class

Let's go ahead and provide this method to our book model. It will come in
handy later. In order do this, we can use one of the provided Valkyrie queries
to find all the resources of a given model. In `app/models/book.rb` add the
following method:

``` ruby
def self.count
  Valkyrie.config.metadata_adapter.query_service.find_all_of_model(model: self).count
end
```

Re-running the controller tests shows that the test is now running successfully
because we've added the missing method, but the test itself is still pending.

Let's get the test running by adding some parameters to the create request.
The scaffold has provided a `let` statement for valid attributes. We can
use that in order to get the tests to execute. Replace the `let` with:

``` ruby
let(:valid_attributes) {
  { title: ["My Book"] }
}
```

Now, when we re-run our spec tests, we should see about 12 failures.
This is because the tests are now getting run instead of skipped.
Looking at the first failure, we should see `undefined method 'create!'
for Book:Class`. Valkyrie doesn't have a `create` method, so we can
implement one in the test. Typically, you would use FactoryBot to do
this, but we'll just create our own method.

Towards the top of the controller spec test, preferably after the `let`
statements, add the following method:

``` ruby
def create_book(attributes)
  Valkyrie.config.metadata_adapter.persister.save(resource: Book.new(attributes))
end
```

This method will server as our `Book.create!` process. Now we can
replace every instance of it with a call to our method. Wherever you see
`book = Book.create! valid_attributes`, replace it with `book =
create_book valid_attributes`.

Re-run your controller test with

    bundle exec rspec spec/controllers/books_controller_spec.rb

There should be about 8 failures, with the top-most error being
`undefined method 'all'`. This is another ActiveRecord-based method that
returns all the instances of our model. Similar to the `count` method
above, we can refactor our Book model to include both the `all` and
`count` methods. The result will look like:

``` ruby
def self.all
  Valkyrie.config.metadata_adapter.query_service.find_all_of_model(model: self).to_a
end

def self.count
  all.count
end
```

Note the `to_a` in our new `all` method. The query service returns
enumerator objects, but we want these to be arrays, to we call the
`to_a` method to convert them. The `count` method then uses this.

Running the spec test will show the error `undefined method 'find' for Book:Class`
so we'll need to update our controller to use one of Valkyrie's query methods
to retrieve the given resource. All we need to do here is update the `set_book`
method as follows:

``` ruby
def set_book
  @book = Valkyrie.config.metadata_adapter.query_service.find_by(id: Valkyrie::ID.new(params[:id]))
end
```

Re-run the specs and we will now get the error: ` undefined method 'errors'`. Looking at the controller,
the `set_book` action is performed prior to the edit request, and is currently
returning a Book object. If remember from the previous part, a Valkyrie::Resource
has no `errors` method, but a change set does. We could change `set_book` to
return a change set instead:

``` ruby
def set_book
  @book = BookChangeSet.new(Valkyrie.config.metadata_adapter.query_service.find_by(id: Valkyrie::ID.new(params[:id])))
end
```

Re-run your spec tests. The edit test should pass now. Are any other tests
failing because of this change? Why or why not?

Now, you should see the new error `undefined method 'save'`

We will need to update our books controller to create a new book using
our change set. Create a new book change set, then validate the parameters
from the form and sync those changes to the change set. Then we can use our
persister to save the new resource. The resulting method should look something
like:

``` ruby
def create
  change_set = BookChangeSet.new(Book.new)
  change_set.validate(book_params)
  change_set.sync
  @book = Valkyrie.config.metadata_adapter.persister.save(resource: change_set.resource)

  respond_to do |format|
    if @book.persisted?
      format.html { redirect_to @book, notice: 'Book was successfully created.' }
      format.json { render :show, status: :created, location: @book }
    else
      format.html { render :new }
      format.json { render json: @book.errors, status: :unprocessable_entity }
    end
  end
end
```

Re-run your specs to verify the tests pass and you'll see another new
error about the `last` method. We can add that to our model as well:

``` ruby
def self.last
  all.last
end
```

At this point there should be about 3 failures in our controller. We'll
finish those up in the next section.

We could also do a quick refactor at this point. `Valkyrie.config.metadata_adapter`
is used twice in the controller. We can memoize this and DRY up our code a little
bit. Create a new private method:

``` ruby
def metadata_adapter
  @metadata_adapter ||= Valkyrie.config.metadata_adapter
end
```

Now we can change the other two calls to `Valkyrie.config.metadata_adapter`
to simply `metadata_adapter`.

Rerun your tests to verify there are no additional failures and continue
to the next part.

### Part 6: Enabling the Remaining Actions in the Controller

#### PUT #update

Add some attributes to update the resource:

``` ruby
let(:new_attributes) {
  { title: ["My Updated Book"] }
}
```

Because our `set_book` action returns a change set, we can simply validate the
new params on our change set directly and then update the resource.

N.B. this may not be the best way to implement this because `@book` is getting
reset. An alternative implementation might be to keep the book and its change set
more separate.

``` ruby
def update
  respond_to do |format|
    if @book.validate(book_params)
      @book.sync
      @book = metadata_adapter.persister.save(resource: @book.resource)
      format.html { redirect_to @book, notice: 'Book was successfully updated.' }
      format.json { render :show, status: :ok, location: @book }
    else
      format.html { render :edit }
      format.json { render json: @book.errors, status: :unprocessable_entity }
    end
  end
end
```

To fix the `undefined method 'reload'` error, replace the lines in the
test with:

``` ruby
updated_book = Valkyrie.config.metadata_adapter.query_service.find_by(id: book.id)
expect(updated_book.title).to eq(["My Updated Book"])
```

This will reload the book from the database and verify that the title
was updated.

You will also need to update the `book_params` method to account for multivalued
fields in the request:

``` ruby
def book_params
  params.require(:book).permit(title: [], author: [], description: [])
end
```

#### DELETE #destroy

The last remaining action in the controller is the delete method.

Running the test will produce an error with `undefined method
'destroy'`. We can update the controller method to use the Valkyrie
persister to delete the resource.

``` ruby
def destroy
  metadata_adapter.persister.delete(resource: @book)
  respond_to do |format|
    format.html { redirect_to books_url, notice: 'Book was successfully destroyed.' }
    format.json { head :no_content }
  end
end
```

Note that `@book` is really a change set, but the persister is only
looking for the resource's id, so either a Valkyrie resource or a change
set should work here.

### Part 7: Testing Out the User Interface

At this point, all the controller actions should work and we should be
able to open a server session and create, edit, and delete books in the
user interface.

Start up a rails server instance

    bunde exec rails s

And visit [http://localhost:3000/books](http://localhost:3000/books)

Using the app, we can perform all the actions, but you'll notice that
none of the attributes are being saved to the resource. You can create a
new book, but you can't save the title, author, or description.

If you look in the logs coming from the server output you should see:

    Unpermitted parameters: :title, :author, :description

While we have updated our controller to allow for multiple fields, the
params hash coming from the form is still sending singular values.
To fix this, we need to add `multiple: true` to each of the form fields
that were auto-generated in `app/views/books/_form.html.erb`. To do
that, update each input like so:

``` ruby
<div class="field">
  <%= form.label :title %>
  <%= form.text_field :title, multiple: true %>
</div>

<div class="field">
  <%= form.label :author %>
  <%= form.text_field :author, multiple: true %>
</div>

<div class="field">
  <%= form.label :description %>
  <%= form.text_area :description, multiple: true %>
</div>
```

Now we should be able to enter values for all our attributes and have
them persist, as well as update and delete each book resource.
