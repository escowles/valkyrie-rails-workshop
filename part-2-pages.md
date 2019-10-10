# Part 2: Adding Pages

This will create a new model for our pages, and add them to a book by means of a "member of" relationship.
In order to save time, we will not focus on tests since we covered that in the previous part. You are welcome to update
the tests as you go along.

## Step 1: Generate a Page Model

Run the Valkyrie generator to create a new page model

     bundle exec rails g valkyrie:resource Page title:string book_id:string

We'll need to edit the `book_id` property to use the Valkyrie::ID type.
Change the attribute to look like:

    attribute :book_id, Valkyrie::Types::ID  

Let's go ahead and create a change set for Page as well:

``` ruby
class PageChangeSet < Valkyrie::ChangeSet
  property :title
  property :book_id
end
```
*Optional* In lieu of a test, you could open up a Rails console session and create a new page. Try adding the page to a
book by using the change set.

## Step 2: Creating a Nested Resource Route

Since pages belong to books, we can create a nested resource route to reflect this. Edit your `config/routes.rb` file as
follows:

``` ruby
resources :books do
  resources :pages
end
```
Check that the routes are correct by running

    bundle exec rake routes

You should see a complete list of routes for pages that are all prefaced by `/books/:book_id`

## Step 3: Build the Form for Adding a New Page

In order to build the form, let's scaffold out the rest of the components needed for our pages. Similar to the books
model, we can use a Rails generator for this. While we're at it, let's go ahead and add a field for a future file upload
that will enable us to upload a binary file for our page.

    bundle exec rails g scaffold_controller Page title:string file:file 

Note that we do not need to include a field for `book_id` because we'll be able to get it from our url parameters thanks
to the nested resources.

Next, edit `app/views/books/show.html.erb` to add a link to the new page form:

``` ruby
<%= link_to 'Add Page', new_book_page_path(@book) %>
```

Restart your Rails server and navigate to the show page for a book. You may need to create a new book in order to do
this. When we click on our new "Add Page" link, there will be a `undefined method 'pages_path'` error. Because we're
using nested routes, our scaffolded code will have some bugs.

Let's fix up the bugs in the generated code. First, we need to update the form url in `app/views/pages/_form.html.erb`
to account for our nested resources path:

``` ruby
<%= form_with(model: [@book, page], local: true) do |form| %>
```

Next, the PagesController will need some help. We can duplicate the same kinds of changes we made in BooksController.
First off, we'll add a callback to load our `@book` object:

``` ruby
before_action :set_book
```

Then we'll add a private method to load and memoize the book from Valkyrie

``` ruby
def set_book
  @book ||= metadata_adapter.query_service.find_by(id: Valkyrie::ID.new(params[:book_id]))
end
```

The additional private methods should look pretty familiar. They'll all be the same kinds of methods we added or changed
in our BooksController.

``` ruby
private

  def set_page
    @page = PageChangeSet.new(metadata_adapter.query_service.find_by(id: Valkyrie::ID.new(params[:id])))
  end

  def page_params
    params.require(:page).permit(title: [], file: [])
  end

  def metadata_adapter
    @metadata_adapter ||= Valkyrie.config.metadata_adapter
  end
```

Refreshing our web page, we'll see one more path error in `app/views/pages/new.html.erb` that can be corrected by
directing the "Back" link to the book's show path.

``` ruby
<%= link_to 'Back', book_path(@book) %>  
```

The "New Page" form should now successfully render, but filling in the form data and clicking 'Create Page' will give us
some errors. We'll need to do the same sorts of changes to PagesController that we did with BooksController. Let's first
update the `create` method to create a new page for us.

``` ruby
  def create
    change_set = PageChangeSet.new(Page.new)
    change_set.validate(page_params.merge(book_id: @book.id))
    change_set.sync
    @page = metadata_adapter.persister.save(resource: change_set.resource)

    respond_to do |format|
      if @page.persisted?
        format.html { redirect_to @book, notice: 'Page was successfully created.' }
        format.json { render :show, status: :created, location: @book }
      else
        format.html { render :new }
        format.json { render json: @page.errors, status: :unprocessable_entity }
      end
    end
  end
```

Note the are a couple of differences between the two controllers. First, we add in the `book_id` parameter to the change
set, and secondly, we redirect back to the book view and not the page.

## Step 3: Listing Pages from a Book

We should now have a page added to our book, but we need to display it. Unlike ActiveRecord, Valkyrie does not have the
same relationship macros such as `:has_many` and `:belongs_to` but it does have a couple of baked-in queries that we can
use to get a listing of related resources.

Because pages are declaring membership to a book, this is an inverse relationship of book to pages. We can create
a `pages` method on our Book model to return a listing of pages using Valkyrie's `find_inverse_references_by`.

``` ruby
def pages
  return [] if id.nil?
  
  Valkyrie.config.metadata_adapter.query_service
    .find_inverse_references_by(resource: self, property: 'book_id')
    .to_a
```

Make that change to `book.rb` and then some view code to `book/show.html.erb` to display the results.

``` ruby
<h2>Pages</h2>
<ul>
  <%= render partial: "page", collection: @book.resource.pages %> 
</ul>
```

Create a new partial `app/views/books/_page.html.erb` with this line:

``` ruby
<li><%= page.title.first %></li>
```

We should now see a listing of each page we add to our book.
