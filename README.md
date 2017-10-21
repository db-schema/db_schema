# DbSchema [![Build Status](https://travis-ci.org/7even/db_schema.svg?branch=master)](https://travis-ci.org/7even/db_schema) [![Gem Version](https://badge.fury.io/rb/db_schema.svg)](https://badge.fury.io/rb/db_schema) [![Join the chat at https://gitter.im/7even/db_schema](https://badges.gitter.im/7even/db_schema.svg)](https://gitter.im/7even/db_schema?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

DbSchema is an opinionated database schema management tool that lets you maintain your DB schema with a single ruby file.

It works like this:

* you create a `schema.rb` file where you describe the schema you want in a special DSL
* you make your application load this file as early as possible during the application bootup in development and test environments
* you create a rake task that loads your `schema.rb` and tell your favorite deployment tool to run it on each deploy
* each time you need to change the schema you just change the `schema.rb` file and commit it to your VCS

As a result you always have an up-to-date database schema. No need to run and rollback migrations, no need to even think about the extra step - DbSchema compares the schema you want with the schema your database has and applies all necessary changes to the latter. This operation is [idempotent](https://en.wikipedia.org/wiki/Idempotence) - if DbSchema sees that the database already has the requested schema it does nothing.

*Currently DbSchema only supports PostgreSQL.*

## Reasons to use

With DbSchema you almost never need to write migrations by hand and manage a collection of migration files.
This gives you a list of important benefits:

* no more `YouHaveABunchOfPendingMigrations` errors - all needed operations are computed from the differences between the schema definition and the actual database schema
* no need to write separate :up and :down migrations - this is all handled automatically
* there is no `structure.sql` with a database dump that constantly changes without reason

But the main reason of DbSchema existence is the pain of switching
between long-running VCS branches with different migrations
without resetting the database. Have you ever switched
to a different branch only to see something like this?

![](https://cloud.githubusercontent.com/assets/351591/17085038/7da81118-51d6-11e6-91d9-99885235d037.png)

Yeah, you must remember the oldest `NO FILE` migration,
switch back to the previous branch,
roll back every migration up to that `NO FILE`,
discard all changes in `schema.rb`/`structure.sql` (and model annotations if you have any),
then switch the branch again and migrate these `down` migrations.
If you already wrote some code to be committed to the new branch
you need to make sure it won't get discarded so a simple `git reset --hard` won't do.
Every migration or rollback loads the whole app, resulting in 10+ seconds wasted.
And at the end of it all you are trying to recall why did you ever
want to switch to that branch.

DbSchema does not rely on migration files and/or `schema_migrations` table in the database
so it seamlessly changes the schema to the one defined in the branch you switched to.
There is no step 2.

Of course if you are switching from a branch that defines table A to a branch
that doesn't define table A then you lose that table with all the data in it.
But you would lose it even with manual migrations.

## Installation

Add this line to your application's Gemfile:

``` ruby
gem 'db_schema', '~> 0.3.rc1'
```

And then execute:

``` sh
$ bundle
```

Or install it yourself as:

``` sh
$ gem install db_schema --prerelease
```

## Usage

First you need to configure DbSchema so it knows how to connect to your database. This should happen
in a file that is loaded during the application boot process - a Rails or Hanami initializer would do.

DbSchema can be configured with a call to `DbSchema.configure`:

``` ruby
# config/initializers/db_schema.rb
DbSchema.configure(
  database: 'my_app_development'
)
```

There is also a Rails' `database.yml`-compatible `configure_from_yaml` method. DbSchema configuration
is discussed in detail [here](https://github.com/7even/db_schema/wiki/Configuration).

After DbSchema is configured you can load your schema definition file:

``` ruby
# config/initializers/db_schema.rb

# ...
load application_root.join('db/schema.rb')
```

This `db/schema.rb` file will contain a description of your database structure
(you can choose any filename you want). When you load this file it instantly
applies the described structure to your database. Be sure to keep this file
under version control as it will be a single source of truth about
the database structure.

``` ruby
# db/schema.rb
DbSchema.describe do |db|
  db.table :users do |t|
    t.primary_key :id
    t.varchar     :email,           null: false, unique: true
    t.varchar     :password_digest, length: 40
    t.timestamptz :created_at
    t.timestamptz :updated_at
  end
end
```

Database schema definition DSL is documented [here](https://github.com/7even/db_schema/wiki/Schema-definition-DSL).

### Production setup

In order to get an always-up-to-date database schema in development and test environments
you need to load the schema definition when your application is starting up. But if you use
an application server with multiple workers (puma in cluster mode, unicorn) in other environments
(production, staging) you may get yourself into situation when different workers simultaneously
run DbSchema code applying the same changes to your database. If this is the case you will need
to disable loading the schema definition in those environments and do that from a rake task called
from your deploy script:

``` ruby
# config/initializers/db_schema.rb
DbSchema.configure(url: ENV['DATABASE_URL'])

if ENV['APP_ENV'] == 'development' && ENV['APP_ENV'] == 'test'
  load application_root.join('db/schema.rb')
end

# lib/tasks/db_schema.rake
namespace :db do
  namespace :schema do
    desc 'Apply database schema'
    task apply: :environment do
      load application_root.join('db/schema.rb')
    end
  end
end
```

Then you just call `rake db:schema:apply` from your deploy script before restarting the app.

If your production setup doesn't include multiple application processes starting simultaneously
(for example if you run one Puma process per docker container and replace containers
successively on deploy) you can go the simple way and just
`load application_root.join('db/schema.rb')` in any environment right from the initializer.
The first puma process will apply the schema while the subsequent ones will see there's nothing
left to do.

## Known problems and limitations

* primary keys are hardcoded to a single NOT NULL integer field with a postgres sequence attached
* array element type attributes are not supported
* precision in all date/time types isn't supported
* no support for databases other than PostgreSQL

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment. To install this gem onto your local machine, run `bundle exec rake install`.

## Contributing

Bug reports and pull requests are welcome on GitHub at [7even/db_schema](https://github.com/7even/db_schema).

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
