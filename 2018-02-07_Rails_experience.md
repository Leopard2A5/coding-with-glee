Ruby on Rails has been around for some time and is in its 5th major version numbers now. I've recently picked it up again for a private project and I've made some observations which I'd like to share.


## Generators
Not an observation in itself, but they save you so much time I just had to mention them. I like you loads, `rails generate`!.


## References and foreign keys
While it's easy to create a reference between model entities with `belongs_to` or `has_many`, if you forget to add `foreign_key: true`, and possibly `null: false`, you'll possibly in for a surprise, as ActiveRecord will not create the corresponding database constraints. I forgot to add these with many of my migrations and only stumbled upon the fact when I tried to create [fixtures](http://guides.rubyonrails.org/testing.html#the-low-down-on-fixtures) for my application. I had something like this:
```yaml
# companies.yml
acme:
  name: ACME

# characters.yml
bugs:
  company: acme
  name: Bugs
```

`acme` and `bugs` are different models and bugs belongs to acme through a foreign key constraint. Rails generates ids by hashing the names of the fixtures, so `acme` may become `51` and `bugs` may become `42`. I had a typo though and ended up with `bugs` referencing a company id that wasn't in the companies table. So I tried to find out why my postgres database didn't spot this error and this led me to the missing foreign key constraints. Easily fixed, but it didn't solve the issue! The database was still allowing invalid references to be inserted! It turns out that Rails inserts fixtures in no particular order, without taking into account dependencies between models, it disables all database triggers, and thus the constraint checks, prior to inserting the fixture data!


## Lesson(s) learned
First of all, when I create a migration that adds a reference to another table, I'll not forget to add `foreign_key: true, null: false` anymore (unless either of them is really not intended, which should be the exception rather than the rule).

Second I've come to the conclusion that I'd rather not use fixtures at all. It is nice to have all your test data visible as a well-structures yaml file, but I'm a pessimistic character and I'd rather not have data inserted into my database with all triggers off. So I'll resort to creating my test data imperatively in my tests. I've come to like Capybara with the webkit driver (basically Rails' "system tests" with a different driver) for this purpose, because it's as end-to-end as you can go without using the browser manually.
