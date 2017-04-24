---
layout: 'post'
title: 'Maintaining Test Databases in Rails, Part 2'
published: false
tags: [rails, testing]
---

So I have a simple question, with a simple answer, that I'm going to ask, so
that I can get to a handful of more complicated questions afterwards.

Since 'Capybara with JavaScript-enabled tests' should not include the
`:rack_test` driver, we should expect to be able to use the transaction strategy
to clean our database between tests with reliable and predictable results.
`:rack_test` should not open up a new connection to the database.

Let's make sure.

Let's set up a Rails app (pre-5.1) with a basic CRUD setup for a `Book` model:

```
$ rails new tests
$ cd tests
```

This generates a Rails 5.0.2 application. Full discrepancy, I'm removing Spring
because I've struggled with it in the past. This *should* be unimportant, but
just in case I figured I'd mention it.

Next, I'm adding 'capybara' to the development and test groups in the Gemfile.

**Gemfile**

```ruby
group :development, :test do
  gem 'byebug', platform: :mri
  gem 'capybara', '~> 2.13.0'
end
```

`bundle install`, and then we can move on to a dummy model we can use to
interact with the test database:

```
$ rails g scaffold book title:string author:string
$ bundle exec rake db:migrate
```

Before we get started with anything interested, let's get some sanity testing
out of the way - not because I don't trust Rails (I do!) but because Stupid
Mistakes Happen and it's nice to have reliable basics to fall back on.

So, what's the simplest way to prove that data is being cleaned between tests?

First I'm going to remove all of the fixtures from test/fixtures/books.yml. I'm
not arguing for or against fixtures at the moment - I just want to test my test
database against an assumed-empty default state, and fixtures would get in the
way of that.

Because I did that, I'm going into the generated BooksControllerTest and
replacing `@book = books(:one)` with `@book = Book.create(title: 'MyString',
author: 'MyString')`. Not a good idea in an actual project, but suits our
purposes here.

Before writing anything, I'm going to check that the test suite runs:

```
$ bundle exec rake

... test output ..

=> 7 runs, 9 assertions, 0 failures, 0 errors, 0 skips
```

We're all good to start. My simplest-passing-proof is as follows in
**test/models/book_test.rb**:

```
require 'test_helper'

class BookTest < ActiveSupport::TestCase
  i_suck_and_my_tests_are_order_dependent!

  test 'A - the truth' do
    Book.create
    assert_equal 1, Book.count
  end

  test 'B - the other truth' do
    assert_equal 0, Book.count
  end
end
```

By ensuring that the tests are order-dependent, we know beyond a shadow of a
doubt that test A runs before test B, and that tells us that the Book which is
(definitely) created in test A does not exist in test B.

Now let's move on to something that gets us closer to proving something a little
more interesting.

I want to show that integration tests, using Capybara configured with the
default `:rack_test` driver, will behave the same way. (Spoiler alert: it will -
but writing the tests to prove it will help us prove something a little less
obvious, so just bear with me.)

Let's set up integration tests. In **test/test_helper**

```ruby
ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('../../config/environment', __FILE__)
require 'rails/test_help'

require 'capybara/rails' # NEW
require 'capybara/dsl' # NEW

class ActiveSupport::TestCase
  ...
end

# NEW
class ActionDispatch::IntegrationTest
  include Capybara::DSL
end
```

And then set up **test/integration/book_integration_test.rb** as follows:

```ruby
require 'test_helper'

class BookIntegrationTest < ActionDispatch::IntegrationTest
  i_suck_and_my_tests_are_order_dependent!

  test 'A - the truth' do
    visit books_path

    assert page.has_content? 'There are no books in the database.'
    refute page.has_content? 'Stranger in a Strange Land'

    visit new_book_path

    fill_in 'Title', with: 'Stranger in a Stranger Land'
    fill_in 'Author', with: 'Robert Heinlein'

    click_button 'Create Book'

    visit books_path

    assert page.has_content? 'Stranger in a Strange Land'
    refute page.has_content? 'There are no books in the database.'
  end

  test 'B - the other truth' do
    visit books_path

    assert page.has_content? 'There are no books in the database.'
    refute page.has_content? 'Stranger in a Strange Land'
  end
end
```

Of course, in order to get 'There are no books in the database.' to show up, we
need to edit the default generated **app/views/books/index.html.erb** file. This
isn't entirely necessary, but it's thorough and that brings me peace.

```erb
...
  <tbody>
    <% if @books.any? %>
      <% @books.each do |book| %>
        ...
      <% end %>
    <% else %>
      There are no books in the database.
    <% end %>
  </tbody>
...
```

The tests should pass at this point, given that they're all saying the same
thing.

But it gets a little weird with Capybara. Let's say we change the test. Let's
replace the last three lines of code in test A:

```
  visit books_path

  assert page.has_content? 'Stranger in a Strange Land'
  refute page.has_content? 'There are no books in the database.'
```

with a simple assertion:

```
  assert_equal 1, Book.count
```

And then replace the entirety of test B with its equivalent:

```
  assert_equal 0, Book.count
```

This passes. But this is exactly the type of thing that would *not* pass using a
different driver. Let's prove it.

First I want to set up my tests so that both the initial integration tests I
wrote (that rely on "seeing" evidense of data in the browser) and the tests that
count the number of books are in the database directly exist. This gets a little
wonky having them all in one file, trying to determine an order, so we're going
to create two integration test files:

1. **test/integration/browser_evidense_test.rb** - will have the integration
tests as they were initially written, "looking" at the browser for evidense of
whether or not our book exists
2. **test/integration/model_query_evidense_test.rb** - will have the integration
tests that query the model to determine whether or not the books exist

From now on, we'll be running each of these tests files individually so that we
can maintain strict control over knowing *exactly* what is getting run and when.

I don't think it's particularly important here to see what's going on in the
browser, so I'm going to use a headless driver. I'll use Poltergeist, because
for me, at this moment, it's the simplest to set up.

```ruby
group :development, :test do
  ...
  gem 'poltergeist'
end
```

```bash
$ bundle install
```

**test/test_helper**

```ruby
require 'capybara/rails'
require 'capybara/dsl'
require 'capybara/poltergeist'

...

class ActionDispatch::IntegrationTest
  include Capybara::DSL

  Capybara.javascript_driver = :poltergeist

  def setup
    Capybara.current_driver = Capybara.javascript_driver
  end

  def teardown
    if Capybara.current_driver == Capybara.javascript_driver
      Capybara.reset_sessions!
      Capybara.use_default_driver
    end
  end
end
```

Without changing the database-cleaning strategy, we can expect the entire test
suite to fail - but don't run them quite yet. Let's take a closer look.

First, run just our unit tests:

```bash
$ bundle exec rake test:models
```

Those pass.

Let's run the integration tests that test the browser:

```bash
$ bundle exec rake test TEST=test/integration/browser_evidense_test.rb
```

On the first run:

```bash
=> # Running:
=>
=> .F
=>
=> Failure:
=> BrowserEvidenseTest#test_B_-_the_other_truth [/path/to/test]:
=> Expected true to not be truthy.
=>
=> ...
=>
=> 2 runs, 5 assertions, 1 failures, 0 errors, 0 skips
```

This fails because our browser at books_path does indeed display 'Stranger in a
Strange Land' in test B, where such a book has no business existing.

Run the test a second time. Both tests will fail this time - because 'Stranger in
a Strange Land' has remained in the test database between runs, as the check in
the first test that indicates that the book should not exist before being
created fails.

If we take away the checks in test A about the book not existing, and assume
there's not a uniqueness validation on a book's title and author, then every
time we run this we're making the test database bigger, filling it up with
more and more copies of the book that never get deleted. Leave this alone,
and our tests will get slow and brittle and stupid. If there *is* a uniqueness
validation on books, the tests will error out, making what we expect to be a
perfectly valid book invalid. If, as a Rails developer, you don't know to look
to the state of the test database between tests in order to debug something like
this, you can lose hours and sanity trying to track it down (source: been there,
done that).

In order to start fresh, we're going to add the line `Book.delete_all` to the
top of our test_helper and run the model tests again. This is an effective way
to wipe our test database for the time being, so we can approach the other
integration test - model_query_evidense - and explore what the way that fails
can tell us. Remove `Book.delete_all` from the test_helper and run the model
query integration test:

```bash
$ bundle exec rake test TEST=test/models/model_query_evidense_test.rb
```

This has the same pattern of behavior, but we can see the number of books in the
test database with the error message for the second failure - we expect 0 books,
we get 1. So if we comment out lines 7 through 10:

```ruby
  test 'A - the truth' do
    # visit books_path
    #
    # assert page.has_content? 'There are no books in the database.'
    # refute page.has_content? 'Stranger in a Strange Land'

    visit new_book_path

    ...
  end
```

We can watch as the number of books in the database gets larger with each
passing run.

So let's make these tests pass.

**Gemfile**

```ruby
group :development, :test do
  ...
  gem 'database_cleaner'
  ...
end
```

```bash
$ bundle install
```

**test/test_helper.rb**

```ruby
...
require 'capybara/poltergeist'
require 'database_cleaner'

...

class ActionDispatch::IntegrationTest
  include Capybara::DSL

  Capybara.javascript_driver = :poltergeist

  def setup
    Capybara.current_driver = Capybara.javascript_driver
    DatabaseCleaner.strategy = :truncate
  end

  def teardown
    Capybara.reset_sessions!
    Capybara.use_default_driver
    DatabaseCleaner.clean
  end
end
```

So we've set up tests inheriting from `ActionDispatch::IntegrationTest` to be
cleaned by `:truncation` after they run. The `TRUNCATE` command will remove all
objects from the database - although it leaves behind remnants that they exist
(so we cannot, for instance, be guaranteed that the first book created in a new
test will have an ID of 1, as we could using `:deletion`).

Remember, before we run these tests for the first time, after having run the
test suite with several times and leaving the database dirty, that if we run,
say, **test/integration/browser_evidense_test.rb** first thing, the following
test will run first:

```ruby
  test 'A - the truth' do
    visit books_path

    refute page.has_content? 'Stranger in a Strange Land'
    ...
  end
```

And it will fail - because the `TRUNCATE` command has not been run yet.



```ruby
require 'test_helper'

class PostIntegrationTest < ActionDispatch::IntegrationTest
  i_suck_and_my_tests_are_order_dependent!

  test "A - the driver will see a post created with Capybara" do
    visit new_post_path

    fill_in 'Title', with: 'ABC Post Title'

    click_button 'Create Post'

    visit root_path

    assert page.has_content? 'ABC Post Title'
  end

  test "B - the driver will see a post created without Capybara" do
    Post.create!(title: 'XYZ Post Title')

    visit root_path

    assert page.has_content? 'XYZ Post Title'
  end

  test "C - all data is cleaned up from previous tests" do
    assert_equal 0, Post.count

    visit root_path

    refute page.has_content? 'ABC Post Title'
    refute page.has_content? 'XYZ Post Title'
  end
end
```

This test passes when running the full suite, which proves that `:rack_test`
manages transactions as well as unit tests do, for three reasons:

1. Because I can see 'ABC Post Title' on the posts#index page after creating the
post titled 'ABC Post Title' in test A, this means that the `:rack_test` driver
can see what's happening to the database that's connected to the driver.
(Proving what we already know.)
2. Because I can see 'XYZ Post Title' on the posts#index page after creating the
post titled 'XYZ Post Title' in test B, this means that the `:rack_test` driver
can see what's happening to the database that's connected to the test suite.
Because we're using `:transaction` as our database cleaning strategy and we know
that connections to the database cannot see into transactions being performed by
other connections, we know that the test and the driver share the same
connection to the database. (Proving the hypothesis.)
3. Because we see no evidense of objects that existed in previous tests in test
"C", we know that our DatabaseCleaner `:transaction` strategy is working as
expected to keep the database clean between tests. (Sanity check to make sure
this test file is operating under the rules we assume it's operating under.)

Now we can test the same functionality using a JavaScript-enabled driver with
Capybara. I'm going to use Capybara's default JavaScript-enabled driver,
Selenium, because it requires no extra setup. Then we'll reconfigure our
test_helper just a little bit:

```ruby
class ActionDispatch::IntegrationTest
  include Capybara::DSL

  # new

  def setup
    DatabaseCleaner.start
    Capybara.current_driver = Capybara.javascript_driver
  end

  def teardown
    DatabaseCleaner.clean

    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end
```

Because we're now switching from a driver that shares the same connection to the
database as the rest of the test suite (`:rack_test`) to one that opens up its
own connection in a separate thread (`:selenium`), we should expect the tests
to fail. Specifically, we should expect:

1. Test "A" should pass, because no tests will run before it that haven't
already proven to clean the database when they're done using transaction as a
cleaning strategy.
2. Test "B" should fail, because `Post.create!(title: 'XYZ Post Title')` is
operating in a different connection than the run that's opened on the line
`visit root_path`.

I'm uncertain whether test "C" will pass or fail. I think that will fail,
because there are two connections to the database at play, both in this test and
the previous one, and one of them will not have been rolled back.

Running the whole test suite:

[image placeholder]

Since I've managed to explain why I think "B" failed, I'm going to focus on "C".

Test "C" failed on line 27, that is, this line:

`assert_equal 0, Post.count`

My guess is that if I comment that line out, test "C" will pass.

And it does.

So far I've only managed to prove what I've already read and posited. But from
here I can definitively answer the question I had when I started this whole
post, which is: Is it reasonable to use `:transaction` as a strategy when using
`:rack_test` as the only driver for your integration tests? Why yes. Yes it is.

But further, what happens if I use `:transaction` for my unit tests and
`:truncation` or `:deletion` for my integration tests? Is mixing these
strategies a bad idea that will only lead to chaos and confusion?

My instinct is to say that I think that if a test suite contains *any*
JavaScript-enabled tests at all, it will introduce the room for intermittant
failures in the test suite transaction is used anywhere in the test suite, even
in places where no separate connection is opened up. That's because the separate
connection introduces the potential for the `ROLLBACK` command to be missed by
other tests, or to wrap itself around only *some* of the data created in any
given JavaScript-aware integration test.

But how to prove it?

First, let's see what happens when we switch our strategy to `:truncation`
(everything should pass).

We're going to change the **test/test_helper.rb** file by getting rid of the line
`DatabaseCleaner.strategy = :transaction` and replacing it with the original
DatabaseCleaner code in both setup methods with the following:

```ruby
def setup
  DatabaseCleaner.strategy = :truncation
  DatabaseCleaner.clean_with :truncation
  DatabaseCleaner.clean
end
```

Then we're removing the line `DatabaseCleaner.clean` from both teardown
methods.

Except they don't. Because the DatabaseCleaner has nothing to do with the fact that Poltergeist has opened its driver with a separate connection to the database. So our tests that try to indicate that `Post.create!(title: 'XYZ Post Title')` will create a post that can be seen when we `visit root_path` are going to fail.

Test "B" fails on line 23, and test "C" fails on line 27. Test "C" failing on Line 27 is the first line of the test:

`assert_equal 0, Post.count`

which means that the database hasn't been cleaned. But if I remove that line the test passes - meaning that the posts that were created by the driver in previous tests *have* been destroyed.

If I remove the order dependence of the test, "C" will sometimes pass and sometimes fail - because sometimes it runs before the `Post.create!` is called (and thus it fails) and sometimes it runs after (and passes). So `DatabaseCleaner.clean` catches the posts created in the driver but not the posts created directly in the test.

But it seems to be catching any and all posts created by the unit tests?

So I commented out test "B" in the integration test, and commented out the lines that set order dependence, in order to test whether or not the unit tests are cleaning themselves properly before running the integration tests. This is my first result (seed 45686):

<img src="file:///Users/wendybeth/Desktop/Desktop/Screen%20Shot%202016-11-16%20at%2010.34.40%20AM.png" />

So the unit tests ran before the integration tests, and test "C" fails because there is still a post in the database when it runs its first assertion. Isolate!

Now I only have:

```ruby
require 'test_helper'

class PostIntegrationTest < ActionDispatch::IntegrationTest
  test "all data is cleaned up from previous tests" do
    puts "Integration"

    assert_equal 0, Post.count
  end
end
```

```ruby
require 'test_helper'

class PostTest < ActiveSupport::TestCase
  test "the truth" do
    puts "Unit"

    Post.create!
    assert_equal 1, Post.count
  end
end
```

That passes whether the Unit prints out before the Integration or not. Grr. OK, but, in this test suite we're not opening a separate connection at all. So, let's uncomment Integration test "A".

<img src="file:///Users/wendybeth/Desktop/Desktop/Screen%20Shot%202016-11-16%20at%2010.42.38%20AM.png" />

Test "C" fails if "A" runs first. Test "C" passes if "A" runs second. Whether the unit test is in there at all, whether the unit tests run before or after the integration tests, makes no difference.

So I've found some very weird behavior, and in trying to understand it came across this blog post [here](http://www.virtuouscode.com/2012/08/31/configuring-database_cleaner-with-rails-rspec-capybara-and-selenium/) which very conveniently mentions that running `:truncation` for JavaScript-enabled tests and `:transaction` for everything else, is, indeed, a convenient way to deal with drivers.

However, let's document the weirdness I'm facing because it's really bothering me:

Test "C" always fails when it runs after test "A". So I did some scouring in my code to figure out why that might be happening. I put a `binding.pry` in `setup` and `teardown` for integration tests and only ran integration tests - order dependence, so I can consistently get the failure I'm looking for.

I get this:

Setup before test "A":

```bash
$ def setup
$   binding.pry
$   ...
$ end
$
$ Post.count
$ => 0
```

Teardown after test "A":

```bash
def teardown
  binding.pry

  DatabaseCleaner.clean

  Capybara.reset_sessions!
  Capybara.use_default_driver
end

$ Post.count
$ => 1
$ DatabaseCleaner.clean
$ => [#<DatabaseCleaner::Base:0x007fcac4ff3de0
	  @db=:default,
	  @orm=:active_record,
	  @strategy=
	  	#<DatabaseCleaner::ActiveRecord::Truncation:0x007fcac696ec70
	  	  @cache_tables=true,
	  	  @connection_class=ActiveRecord::Base,
	  	  @db=:default,
	  	  @only=nil,
	  	  @pre_count=nil,
	  	  @reset_ids=nil,
	  	  @tables_to_exclude=["schema_migrations"]>>]
$ Post.count
$ => 0
```

So that's... what should be happening. ...

So I exit out of that and I should get the setup method for test "B" next. And I do.

```bash
$ Post.count
$ => 1
```

:(

Close out of that, and:

```bash
$ test "C - all data is cleaned up from previous tests" do
$   puts "Integration C"
$
$   binding.pry
$   ...
$  end
$
$ Post.count
$ => 1
```

And exist Pry session, and... in teardown. Where Post.count is still 1 until running DatabaseCleaner.clean.

Then test failure.

*Why is there a post that exists in Integration Test "C" when the Post table is empty in the teardown after integration test "A"?*

And I think I may have figured it out.

In setup, right before test "C", I ran `DatabaseCleaner.clean` *after* setting the Javascript driver, and then the Database is clean. So when running `DatabaseCleaner.clean` in `teardown`, I'm running it in the one ...

But shouldn't DatabaseCleaner.clean, using the `:truncation` strategy, empty the database and therefore it shouldn't *matter* which connection to the database is checking its contents? It should be empty.

And it was because I hadn't set `self.use_transactional_fixtures` to false. Because Rails automatically wraps all tests in a transaction and that was getting in DatabaseCleaner's way. So:

```ruby
class ActionDispatch::IntegrationTest
  include Capybara::DSL
  self.use_transactional_fixtures = false

  def setup
    DatabaseCleaner.strategy = :truncation

    Capybara.javascript_driver = :poltergeist
    Capybara.current_driver = Capybara.javascript_driver
    Capybara.server_port = "5000"
    Capybara.run_server = true

    DatabaseCleaner.clean
  end

  def teardown
    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end
```

I also added `self.use_transactional_fixtures = false` to `ActiveSuport::TestCase`. Now my tests pass.

I go in and uncomment them all... and re-instate the order-dependence. I expect these tests to pass now. And they do. I just got all of the setup wrong to begin with. Now we can move on. We know that using `:truncation` for tests keeps out database clean. *phew*

Let's *finally* get to question 3:

**Q3:** What happens if I use `:transaction` for my unit tests and `:truncation` or `:deletion` for my integration tests?

I linked to a [blog post](http://www.virtuouscode.com/2012/08/31/configuring-database_cleaner-with-rails-rspec-capybara-and-selenium/) by Avdi Grimm earlier that, in fact, recommends this strategy. And that feels good enough to me. But I'm gonna go ahead and run that strategy through my little experiment here, just so I can prove definitively that this will work.

So, I'm changing my test setup:

```ruby
DatabaseCleaner.clean_with :truncation # before our test suite runs, ensure the database is empty

class ActiveSupport::TestCase
  fixtures :all
  self.use_transactional_fixtures = false

  def setup
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
  end

  def teardown
    DatabaseCleaner.clean
  end
end

class ActionDispatch::IntegrationTest
  include Capybara::DSL
  self.use_transactional_fixtures = false

  def setup
    DatabaseCleaner.strategy = :truncation

    Capybara.javascript_driver = :poltergeist
    Capybara.current_driver = Capybara.javascript_driver
    Capybara.server_port = "5000"
    Capybara.run_server = true

    DatabaseCleaner.clean
  end

  def teardown
    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end
```

And our tests should pass. And even though it makes no real difference with so few tests, they should be faster, too.

And they do pass. Although they do take almost 8 seconds to run which seems...problematic. I'll go ahead and try to use the `:deletion` strategy. No noticeable difference in speed but still passes.

So, here's an old, "default" test/acceptance helper combo that we've used for some of our production applications:

**test/test_helper.rb**

```ruby
require 'rubygems'
require 'simplecov'

SimpleCov.merge_timeout 3600
SimpleCov.start

require 'minitest/autorun'
require 'minitest/pride'
require 'fakeweb'
require 'database_cleaner'
require 'awesome_print'

ENV['RAILS_ENV'] = 'test'

require File.expand_path('../../config/environment', __FILE__)

require 'fabrication'
Fabrication.configure do |config|
  fabricator_path = "test/fabricators"
end

module TestHelper
  def setup
    setup_database
  end

  def setup_database
    DatabaseCleaner.strategy = :truncation
    DatabaseCleaner.clean_with :truncation
    DatabaseCleaner.clean
  end

  def teardown
  end

  ...
end
```

**test/acceptance_helper.rb**

```ruby
require 'test_helper'
require 'rack/test'
require 'mocha/setup'
require 'capybara/rails'
require 'capybara/dsl'
require 'capybara/screenshot'
require 'capybara/poltergeist'

module AcceptanceHelper
  include Capybara::DSL
  include Rack::Test::Methods
  include TestHelper
  include Rails.application.routes.url_helpers

  Capybara.register_driver :poltergeist do |app|
    Capybara::Poltergeist::Driver.new(app, { :js_errors => false })
  end

  def setup
    setup_database
  end

  def setup_capybara
    Capybara.javascript_driver = :poltergeist
    Capybara.current_driver = Capybara.javascript_driver
    Capybara.server_port = "5000"
    Capybara.run_server = true
  end

  def teardown
    Capybara.reset_sessions!
    Capybara.use_default_driver
  end
end
```

So I can now change this to reflect my new, better knowledge, and define setup_database differently in TestHelper and AcceptanceHelper.

So tests in this project, as is, run:

1. 47.374245s, 1.6465 runs/s, 4.4328 assertions/s.
2. 74.221272s, 1.0509 runs/s, 2.8294 assertions/s.
3. 51.232530s, 1.5225 runs/s, 4.0990 assertions/s.
4. 78.104231s, 0.9987 runs/s, 2.6887 assertions/s.
5. 82.777551s, 0.9423 runs/s, 2.5369 assertions/s.

average:
66.7419658, 1.23218 runs/s, 3.31736 assertions/s

78 runs, 210 assertions, 0 failures, 0 skips

Changing the unit tests to use `:transaction`:

1. 13.642502s, 5.7174 runs/s, 15.3931 assertions/s
2. 13.440451s, 5.8034 runs/s, 15.6245 assertions/s
3. 12.407042s, 6.2868 runs/s, 16.9259 assertions/s
4. 12.457002s, 6.2615 runs/s, 16.8580 assertions/s
5. 13.200422s, 5.9089 runs/s, 15.9086 assertions/s

average:
13.0294838, 5.9956 runs/s, 15.94202 assertions/s

78 runs, 210 assertions, 0 failures, 0 skips

...

So. Yeah. That made a difference.



---

insights: after change, using truncation

1. 357.791608s
2. 362.551628s
3. 322.200327s
4. 342.602377s
5. 315.946896s

insights: after change, using deletion

1. 318.154178s
2. 319.817280s
3. 368.155835s
4.
