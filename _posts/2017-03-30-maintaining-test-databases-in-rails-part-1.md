---
layout: 'post'
title: 'Maintaining Test Databases in Rails, Part 1'
tags: [rails, testing]
---

## Maintaining Test Databases

Assumed tools:

+ Rails (ActiveRecord) - prior to 5.1
+ Minitest (the principles should apply to RSpec - the syntax won't)
+ Capybara

[DatabaseCleaner](https://github.com/DatabaseCleaner/database_cleaner) is a
well-known gem that helps Rails developers manage database access from their
test suites. I've been using it for a couple of years but wanted to take a
deeper look into the specifies of its usage, so I did some research and
experimenting in order to clear up a few gaps in my knowledge. I would like to
share some of that process for review and in hopes of clarifying some of the
more confusing points of test-database management in Rails for others who, like
me, may have missed some of the finer details.

This post is the first in a series that explores test database management in
Rails. It provides a brief overview of the three database-cleaning strategies
that can be used by DatabaseCleaner (with ActiveRecord, specifically) and an
explanation for when and why we would ever need to look beyond the fastest,
most convenient database-cleaning strategy, `:transaction`.

### Section 1: DatabaseCleaner cleaning strategies

I came up with the idea to write this post when I came upon the following two
lines of code:

```ruby
DatabaseCleaner.strategy = :truncation
DatabaseCleaner.clean
```

I realized I didn't really understand what `:truncation` meant in this context.
So I set out to learn.

A strategy for DatabaseCleaner outlines how the database is wiped of test data
between individual tests, so that each test starts with a clean (more
importantly: *controlled*) slate.

There are three strategies that DatabaseCleaner provides, each named for a SQL
command that manages the removal of data from a relational database. There's
`:transaction`, `:truncation`, and `:deletion`.

`:transaction`, according to the DatabaseCleaner documentation, is the *fastest*
strategy.

[Transactions](https://en.wikipedia.org/wiki/Database_transaction) are a useful
concept in the management of databases. They entail a process of wrapping a series
of database commands in a block and then ensuring that each and every command in the
block is successful before committing or, should even one fail, undoing all of them
at once. This is where the `ROLLBACK` command comes in. It undoes everything within
the transaction, restoring the database to its pre-transaction state.

DatabaseCleaner takes advantage of the `ROLLBACK` command between tests, by
wrapping each one in a transaction (started with the line
`DatabaseCleaner.start`, usually called in the `setup` method or in a `before`
block), and then rools that transaction back before the start of the next test
(with `DatabaseCleaner.clean`, usually called in the `teardown` method or in an
`after` block).

I have come across the concept of transactions many times over the past few years,
as they are a useful and prevalent tool when creating and managing web applications.
Because I am self-taught, I often come upon strange little gaps in my knowledge,
and when it came to the `:truncation` and `:deletion` strategies, I had little
context beyond what I've explained to understand what was going on.

The most immediate source of information, which will definitely be relevant to
DatabaseCleaner, is in the DatabaseCleaner documentation itself. Because I'm sure
there is a rabbit hole of knowledge that just searching for 'truncation' or 'deletion'
independently of DatabaseCleaner could lead me down, I'd rather follow the resources
that the maintainers of the DatabaseCleaner gem find most relevant to the usage of their gem.

> For the SQL libraries, the fastest option will be to use `:transaction` as
transactions are simply rolled back. If you can use this strategy you should.
However, if you wind up needing to use multiple database connections in your
tests (i.e. your tests are in a different process than your application)
then using this strategy becomes a bit more difficult. You can get around the
problem in a number of ways.
>
> One common approach is to force all processes to use the same database connection
> (common ActiveRecord hack) however this approach has been reported to result in
> non-deterministic failures.
>
> An easier, but slower solution is to use the `:truncation` or `:deletion` strategy.

So the problem addressed by `:truncation` (or `:deletion`) is caused by the
usage of JavaScript-enabled drivers used by Capybara, typically in
integration/acceptance/system tests.

When a new connection to the database is opened up, that connection cannot see
transaction states that are currently being managed by other connections. In the
case of an integration/system/acceptance test that uses a JavaScript-enabled
driver, the driver opens up a browser with a different connection to the test
database from the test server itself. This means that data created in the test
by the server (say you call `Post.create` in a test) and data created in the
test by the browser (calling `click_button 'Submit'` after filling out a new
Post form) are not handled together by a transaction wrapped around an
individual test. The transaction will see (and thus rollback) the data created
by the driver but not the data created by the server, and thus data will float
around in a fairly unpredictable manner between tests, causing all sorts of havoc.

So knowing why `:transaction` can be a problematic strategy to use with integration
tests, then, let's move on to `:truncation` and `:deletion`.

This [answer on StackOverflow](http://stackoverflow.com/questions/11419536/postgresql-truncation-speed/11423886#11423886)
that's linked to in the DatabaseCleaner documentation is as good a starting point
as any. This sentence, in particular, gives away a lot of useful information with
relatively few words:

> A `TRUNCATE` gives you a completely new table and indexes as if they were just
> `CREATE`d. It's like you deleted all the records, reindexed the table and did
> a vacuum full.

This was my 'ah-ha!' moment, that lead to the realization that `TRUNCATE` and
`DELETE` (like `TRANSACTION` and `ROLLBACK`) are just SQL commands.

To summarize the most useful bits of the StackOverflow answer, which compares
and contrasts `DELETE` and `TRUNCATE` (providing a definition through inference
about what each command actually is on its own):

+ The `DELETE` statement leaves behind some amount of cruft, such as records
  that rows existed at certain indices even after the rows are gone - the
  next object's unique ID will be however many objects *are* and/or *have been*
  in the database, plus one (if there are four objects currently in the
  database and two other objects were deleted awhile ago, the next ID will be 7)
+ `TRUNCATE` wipes the table completely so that it starts from ground zero -
  the next object's unique ID will be 1, no matter how many objects existed or
  existed and then were deleted before the `TRUNCATE` command was run

Note how much we now know about `TRUNCATE`, and how much we can extrapolate,
just from the material right in front of us. `DELETE` and `TRUNCATE` will wipe
the database completely clean, and neither command relies on transaction state
to do so. This means that the database state can be read the same from every
connection. Data is deleted when it should be, and all connections are aware
of the current state of the database when there are no transactions running.

The transaction/rollback process is faster, because it never commits anything to
the database.

A "formal" definition of the `TRUNCATE` command from
[this Wikipedia article](https://en.wikipedia.org/wiki/Truncate_(SQL)) helps to
connect the definition gained through inference about how `:truncation` operates
as a strategy for DatabaseCleaner in our tests to the wider world of general
database management:

> In SQL, the `TRUNCATE TABLE` statement is a Data Definition Language (DDL)
> operation that marks the extents of a table for deallocation (empty for reuse).
> **The result of this operation quickly removes all data from a table**, typically
> bypassing a number of integrity enforcing operations.

So to summarize:

1. Using `strategy = :transaction` will wrap each test in a database
transaction, and then rollback any changes that were made. This is fast, because
it never commits anything to the actual test database, but dangerous when there
are multiple points of access to the database that cannot see one another (such
as when we are using Capybara with a JavaScript-enabled driver).
2. Using `strategy = :truncation` removes all data from all of the tables in the
test database between each test, but does not leave the database in a 'clean'
state - new record IDs will be generated with awareness that there used to be
rows in the table (you'll never get another record with id=1), for example
3. Using `strategy = :deletion` wipes the database *completely* clean - removing
all data and all memory of data from the database, starting it over from scratch
as if it's never been touched

I have more posts on the subject of test database management, but since learning
so much from writing them, I've realized I need to go back and maybe
restructure/rewrite them a little bit.

Some of the questions I had and am attempting to address in the following posts
are:

1. If we have an application that uses the `:rack_test` driver for all of its
integration tets, can we safely get away with using `:transaction` as a strategy
for managing the test database?
2. What happens when we use `:transaction` for unit tests and `:truncation` or
`:deletion` only for integration tests? Is using different strategies through
the same test suite in a controlled fashion a recipe for chaos, or an efficient
way to balance data integrity with test speed?
3. When should we distinguish between the `:truncation` and `:deletion` strategies?
4. What has Rails 5.1 done to address these issues (as you can read in [this
blog post](http://weblog.rubyonrails.org/2017/2/23/Rails-5-1-beta1/), it has)?
