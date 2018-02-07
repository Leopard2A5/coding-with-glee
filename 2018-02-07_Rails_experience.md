Ruby on Rails has been around for some time and is in its 5th major version numbers now. I've recently picked it up again for a private project and I've made some observations which I'd like to share.


## Generators
Not an observation in itself, but they save you so much time I just had to mention them. I like you loads, `rails generate`!. However, as I'm going to point out in this post, be sure to check what is generated and edit it according to your needs.


## References and foreign keys
While it's easy to create a reference between model entities with `belongs_to` or `has_many`, if you forget to add `foreign_key: true`, and possibly `null: false`, you'll possibly be in for a surprise, as ActiveRecord will not create the corresponding database constraints. I forgot to add these with many of my migrations and only stumbled upon the fact when I tried to create [fixtures](http://guides.rubyonrails.org/testing.html#the-low-down-on-fixtures) for my application. I had something like this:
```yaml
# companies.yml
acme:
  name: ACME

# characters.yml
bugs:
  company: acme
  name: Bugs
```

`acme` and `bugs` are different models and bugs belongs to acme through a foreign key constraint. When using fixtures, Rails generates ids by hashing the names of the fixtures, so `acme` may become `51` and `bugs` may become `42`. I had a typo though and ended up with `bugs` referencing a company id that wasn't in the companies table. So I tried to find out why the referential integrity in my Postgres database didn't spot this error and this led me to the missing foreign key constraints. I forgot to add the foreign key option to many of my migrations!

Easily fixed, but it didn't solve the issue! The database was still allowing invalid references to be inserted! It turns out that Rails inserts fixtures in no particular order, without taking into account dependencies between models. It can only do this by disabling database triggers, and thus the constraint checks, prior to inserting the fixture data, which is exactly what it does. I found this out by looking at the logs my Postgres docker container was putting out.


## Lesson(s) learned
First of all, when I create a migration that adds a reference to another table, I'll not forget to add `foreign_key: true, null: false` anymore (unless either of them is really not intended, which should be the exception rather than the rule).

Second I've come to the conclusion that I'd rather not use fixtures at all. It is nice to have all your test data visible as a well-structures yaml file, but I'm a pessimistic character and I'd rather not have data inserted into my database with all triggers off. So I'll resort to creating my test data imperatively in my tests.

I know: it's not as neat as having an overview of all static test data in yaml files, but the missing referential integrity is a sort of deal breaker for me. Also with a few helper functions i can create very concise and ergonomic tests imperatively.

I've come to like [Capybara](https://teamcapybara.github.io/capybara/) with the webkit driver (basically Rails' ["system tests"](http://guides.rubyonrails.org/testing.html#system-testing) with a different driver) for this purpose, because it's as end-to-end as you can get without using the browser manually.

End-to-end is another topic I'd like to highlight here. When testing, I prefer the method that is as end-to-end as possible. Of course that increases the time it needs to test on the CI/CD pipeline but I think it's worth it. That's not to say that unit tests don't have a reason to be there, on a component level I think they're very useful and I use them for TDD.

When it comes to integration tests though, I'd rather write end-to-end tests to test the application as a whole than do piecemeal integration testing. I'm not saying integration testing doesn't have its place, but when in doubt I'll go end-to-end.

Consider the following example of system testing:
```ruby
class FoobarTest < ApplicationSystemTestCase
  test 'requires login' do
    visit foobars_path
    assert_current_path sign_in_path
  end

  test 'login' do
    visit sign_in_path
    fill_in 'Email', with: 'username'
    fill_in 'Password', with: 'password'
    click_button 'Submit Session'
    assert_current_path root_path
    assert_text "Signed in as: username"
  end
end
```

The first tests asserts that the user is redirected to the login page when requesting a path that requires a user to be logged in. The second tests that the user can actually log in. In this case it fills in the credentials, hits the submit button and then asserts that the user is redirected to the root path and that the text "Signed in as: username" is present in the page.

## A technical issue with Capybara and the webkit driver
On my dev environment running simple tests with Capybara and webkit was straight forward, but in my CI/CD pipeline Capybara failed to start the test server. I figured out that the webkit driver, although it's headless, still needs an X server to function. So the command to run the tests changed from `rails test:system` to `xvfb-run -a rails test:system`, using the [xvfb](https://en.wikipedia.org/wiki/Xvfb) X server implementation.
