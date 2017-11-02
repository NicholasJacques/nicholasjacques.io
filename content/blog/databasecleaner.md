---
title: "You (probably) don't need DatabaseCleaner"
date: 2017-10-31T21:18:58-06:00
draft: false
categories: ["testing"]
tags: ["rails", "ruby", "databasecleaner", "rspec", "capybara", "testing", "tdd"]
---


If you're like me and use a combination of [RSpec](https://github.com/rspec/rspec-rails) and [Capybara](https://github.com/teamcapybara/capybara) to test your Rails applications, then you are probably familiar with [DatabaseCleaner](https://github.com/DatabaseCleaner/database_cleaner). DatabaseCleaner is a gem that is just about ubiquitous in Rails projects and tutorials. Until recently, I never questioned its omnipresence and dutifully added it too all of my Rails projects. Sure, Rails already includes transactional tests by default, (it's the line you set to 'false' when configuring DatabaseCleaner) but reasons!

Prior to the release of Rails 5.1, there was one very valid reason for continuing to use DatabaseCleaner. Setting `config.use_transactional_fixtures = true` didn't properly wipe the database when testing with some drivers. If you wanted to write a feature test that interacts with client-side JavaScript, you'd have to use [Selenium](https://github.com/SeleniumHQ/selenium), [Capybara-webkit](https://github.com/thoughtbot/capybara-webkit), or [Poltergeist](https://github.com/teampoltergeist/poltergeist) to run your test. Unlike the default driver, [:rack_test](https://github.com/rack-test/rack-test), these drivers run tests against an actual HTTP server. This is problematic because Capybara will spin up a server in a seperate thread from your test. Database transactions are not shared across threads so any changes to your database that occur in these tests will not be properly rolled back after the test is completed.

You probably solved this problem with a solution that looks something like this:

```ruby
RSpec.configure do |config|
  config.before(:suite) do
    begin
      DatabaseCleaner.start
      FactoryGirl.lint
    ensure
      DatabaseCleaner.clean_with(:truncation)
    end
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, :js => true) do
    DatabaseCleaner.strategy = :truncation
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
end
```

This DatabaseCleaner configuration wraps each test in a transaction by default and then rolls back the transaction when the test example is completed. When a test is marked with a `:js => true` flag, it will be run with a test driver other than `:rack_test`. In this case, we tell DatabaseCleaner to clean the database using truncation, since the Capybara will be interacting with your application in a seperate thread from the transaction, as I mentioned earlier. We choose to make cleaning with transactions the default strategy because rolling back a transaction is typically faster than truncating your database, which uses a `TRUNCATE TABLE` statement to wipe the database clean after the test has been run. This configuration solves our threading issue and ensures the test database is always in a clean state for each test example.


With the release of Rails 5.1, the database connection is now [shared between the application and test threads](https://github.com/rails/rails/pull/28083). This effectively renders DatabaseCleaner obsolete. Tests using a non-default driver will be cleaned by simply setting `config.use_transactional_fixtures = true`, allowing you to safely remove the DatabaseCleaner gem from your application.

---

*`use_transactional_fixtures` was changed to `use_transactional_tests` in Rails 5.1 to [clarify its functionality](https://blog.bigbinary.com/2016/05/26/rails-5-renamed-transactional-fixtures-to-transactional-tests.html). If you are using RSpec, then it still appears as `use_transactional_fixtures` in your `rails_helper.rb` file.
