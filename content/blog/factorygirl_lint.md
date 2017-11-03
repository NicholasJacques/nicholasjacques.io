---
title: "Using FactoryGirl and Transactional Testing"
date: 2017-11-02T00:51:39-06:00
draft: false
categories: ["testing"]
tags: ["rails", "ruby", "databasecleaner", "rspec", "testing"]
---

### The Problem

In the process of removing DatabaseCleaner from my Rails 5.1 project, I ran into one complication involving the FactoryGirl gem. FactoryGirl is a fixture replacement tool that lets me easily create objects in my tests. You'll notice that I call `FactoryGirl.lint` in a `before(:suite)` block in my old Database cleaner configuration:

```ruby
# spec/support/database_cleaner.rb

RSpec.configure do |config|
  config.before(:suite) do
    begin
      DatabaseCleaner.start
      FactoryGirl.lint # <--- HERE
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

`FactoryGirl.lint` is a method provided by the FactoryGirl gem to test the validity of your factories before your test suite is run. Using this method can save you a lot of time and anguish because it will alert you to any invalid factories that you might have before running your entire test suite. It works by building and saving each of your defined factories and throws a helpful error message if any of them happen to be invalid. You'll also notice that I call `DatabaseCleaner.clean_with(:truncation)` after running `FactoryGirl.lint` to clean all of these test objects from the database, ensuring a clean slate for my test suite.

When I removed the DatabaseCleaner gem from my project, I initially moved `FactoryGirl.lint` to my FactoryGirl configuration file like so:

```ruby
# spec/support/factory_girl.rb

RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
  config.before(:suite) do
    FactoryGirl.lint
  end
end
```

Unfortunately, this did not work. Since I am now relying solely on Rails' built in transactional tests, I cannot easily clean these test objects out of the database before my test suite runs. With this configuration, I'm starting each test run with a dirty database, which is no good!

One potential solution to this problem is to write a spec specifically for calling `FactoryGirl.lint`.

```ruby
  RSpec.describe "FactoryGirl" do
    it "has valid factories" do
      FactoryGirl.lint
    end
  end
```

This would allow me to take advantage of Rails' transactional tests, which would rollback changes to database after the test completes. Unfortunately, this has a flaw. With `FactoryGirl.lint` wrapped in a spec, I can not be sure of when my factories will be linted in the test suite. Half my test suite could run before finding out I have an invalid factory, which defeats the purpose of linting my factories altogether.

### The Solution

If all of my tests can be wrapped in a database transaction, then why can't I wrap `FactoryGirl.lint` in a transaction as well, and roll back the changes after it checks my factories?

```ruby
# spec/support/factory_girl.rb

RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods

  config.before(:suite) do
    conn = ActiveRecord::Base.connection
    conn.transaction do
      FactoryGirl.lint
      raise ActiveRecord::Rollback
    end
  end
```

After doing a little research, I learned how to do just that. With this configuration, I call `FactoryGirl.lint` within a transaction, and then rollback that transaction before any changes are committed to the database. Success!

### An Improvement

One downside to linting your factories is it can negatively affect the performance of your test suite. If you are working on a larger project with a lot of factories, you might notice a significant lag before your suite runs. This can be especially annoying if you are just running one test that has to wait for FactoryGirl to lint all of your factories, whether they are present in that test or not.

One potential solution, that I found while reading [this](https://github.com/thoughtbot/factory_bot/issues/923) issue thread, is to skip factory linting if you are only running one test. This is pretty easy to implement in my existing code:

```ruby
# spec/support/factory_girl.rb

RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods

  config.before(:suite) do
    conn = ActiveRecord::Base.connection
    conn.transaction do
      FactoryGirl.lint unless config.files_to_run.one?
      raise ActiveRecord::Rollback
    end
  end
```

Another solution changes the use of `FactoryGirl.lint` slightly. The maintainers of FactoryGirl recommend not running `FactoryGirl.lint` in a `before(:suite)` to avoid negatively impacting the performance when running a single test. They instead recommend building a rake task to check your factories at your own discretion. The example rake task that they provide makes use of the DatabaseCleaner gem but this can easily be changed to make use of a database transaction instead:

```ruby
# lib/tasks/factory_girl.rake

namespace :factory_girl do
  desc "Verify that all FactoryBot factories are valid"
  task lint: :environment do
    if Rails.env.test?
      puts "Checking your factories..." 

      conn = ActiveRecord::Base.connection
      conn.transaction do
        FactoryGirl.lint
        raise ActiveRecord::Rollback
      end

      puts "...Factory check completed"
    else
      system("bundle exec rake factory_girl:lint RAILS_ENV='test'")
      exit $?.exitstatus
    end
  end
end
```

This allows you to run `rake factory_girl:lint` whenever you make changes to your factories or models.

---
#### Further Reading
In researching for this post, I found some informative blogs on using FactoryGirl.

* [Working Effectively with Data Factories Using FactoryGirl](https://semaphoreci.com/community/tutorials/working-effectively-with-data-factories-using-factorygirl)
* [Why I don't like factory_girl](https://semaphoreci.com/community/tutorials/working-effectively-with-data-factories-using-factorygirl)
* [Speed Up Tests by Selectively Avoiding Factory Girl](https://robots.thoughtbot.com/speed-up-tests-by-selectively-avoiding-factory-girl)

---

*Thoughtbot recently [changed](https://robots.thoughtbot.com/factory_bot) the name of FactoryGirl to FactoryBot. For the sake of familiarity, I have stuck with the old name for now. I plan to edit this post in the future to reflect the new name once it has gained wider adoption.