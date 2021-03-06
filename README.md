# sequel-activerecord_connection

This is an extension for [Sequel] that allows it to reuse an existing
ActiveRecord connection for database interaction.

This can be useful if you're using a library that uses Sequel for database
interaction (e.g. [Rodauth] or [rom-sql]), but you want to avoid creating a
separate database connection. Or if you're transitioning from ActiveRecord to
Sequel, and want the database connection to be shared.

It works on ActiveRecord 4.2+ and fully supports PostgresSQL, MySQL and SQLite
adapters, both the native ones and JDBC (JRuby). There is attempted suppport
for [Oracle enhanced] and [SQL Server] Active Record adapters (`oracle` and
`tinytds` in Sequel). Other adapters might work too, but their integration
hasn't been tested.

## Installation

Add this line to your application's Gemfile:

```rb
gem "sequel-activerecord_connection", "~> 1.0"
```

And then execute:

```sh
$ bundle install
```

Or install it yourself as:

```sh
$ gem install sequel-activerecord_connection
```

## Usage

Assuming you've configured your ActiveRecord connection, you can initialize the
appropriate Sequel adapter and load the `activerecord_connection` extension:

```rb
require "sequel"

DB = Sequel.postgres(extensions: :activerecord_connection)
```

Now any Sequel operations that you make will internaly be done using the
ActiveRecord connection, so you should see the queries in your ActiveRecord
logs.

```rb
DB.create_table :posts do
  primary_key :id
  String :title, null: false
  Stirng :body, null: false
end

DB[:posts].insert(
  title: "Sequel::ActiveRecordConnection",
  body:  "Allows Sequel to reuse ActiveRecord's connection",
)
#=> 1

DB[:posts].all
#=> [{ title: "Sequel::ActiveRecordConnection", body: "Allows Sequel to reuse ActiveRecord's connection" }]

DB[:posts].update(title: "sequel-activerecord_connection")
#=> 1
```

The database extension supports `postgresql`, `mysql2` and `sqlite3`
ActiveRecord adapters, just make sure to initialize the corresponding Sequel
adapter before loading the extension.

```rb
Sequel.postgres(extensions: :activerecord_connection) # for "postgresql" adapter
Sequel.mysql2(extensions: :activerecord_connection)   # for "mysql2" adapter
Sequel.sqlite(extensions: :activerecord_connection)   # for "sqlite3" adapter
```

If you're on JRuby, you should be using the JDBC adapters:

```rb
Sequel.connect("jdbc:postgresql://", extensions: :activerecord_connection) # for "jdbcpostgresql" adapter
Sequel.connect("jdbc:mysql://", extensions: :activerecord_connection)      # for "jdbcmysql" adapter
Sequel.connect("jdbc:sqlite://", extensions: :activerecord_connection)     # for "jdbcsqlite3" adapter
```

### Transactions

This database extension keeps the transaction state of Sequel and ActiveRecord
in sync, allowing you to use Sequel and ActiveRecord transactions
interchangeably (including nesting them), and have things like ActiveRecord's
and Sequel's transactional callbacks still work correctly.

```rb
ActiveRecord::Base.transaction do
  DB.in_transaction? #=> true
end
```

Sequel's transaction API is fully supported:

```rb
DB.transaction(isolation: :serializable) do
  DB.after_commit { ... } # executed after transaction commits
  DB.transaction(savepoint: true) do # creates a savepoint
    DB.after_commit(savepoint: true) { ... } # executed if all enclosing savepoints have been released
  end
end
```

When registering transaction hooks, they will be registered on Sequel
transactions when possible, in which case they will behave as described in the
[Sequel docs][sequel transaction hooks].

```rb
# Sequel: An after_commit transaction hook will always get executed if the outer
# transaction commits, even if it's added inside a savepoint that's rolled back.
DB.transaction do
  ActiveRecord::Base.transaction(requires_new: true) do
    DB.after_commit { puts "after commit" }
    raise ActiveRecord::Rollback
  end
end
#>> BEGIN
#>> SAVEPOINT active_record_1
#>> ROLLBACK TO SAVEPOINT active_record_1
#>> COMMIT
#>> after commit

# Sequel: An after_commit savepoint hook will get executed only after the outer
# transaction commits, given that all enclosing savepoints have been released.
DB.transaction(auto_savepoint: true) do
  DB.transaction do
    DB.after_commit(savepoint: true) { puts "after commit" }
    raise Sequel::Rollback
  end
end
#>> BEGIN
#>> SAVEPOINT active_record_1
#>> RELEASE SAVEPOINT active_record_1
#>> COMMIT
#>> after commit
```

In case of (a) adding a transaction hook while Active Record holds the
transaction, or (b) adding a savepoint hook when Active Record holds any
enclosing savepoint, Active Record transaction callbacks will be used instead
of Sequel hooks, which have slightly different behaviour in some circumstances.

```rb
# ActiveRecord: An after_commit transaction callback is not executed if any
# if the enclosing savepoints have been rolled back
ActiveRecord::Base.transaction do
  DB.transaction(savepoint: true) do
    DB.after_commit { puts "after commit" }
    raise Sequel::Rollback
  end
end
#>> BEGIN
#>> SAVEPOINT active_record_1
#>> ROLLBACK TO SAVEPOINT active_record_1
#>> COMMIT

# ActiveRecord: An after_commit transaction callback can be executed already
# after a savepoint is released, if the enclosing transaction is not joinable.
ActiveRecord::Base.transaction(joinable: false) do
  DB.transaction do
    DB.after_commit { puts "after commit" }
  end
end
#>> BEGIN
#>> SAVEPOINT active_record_1
#>> RELEASE SAVEPOINT active_record_1
#>> after commit
#>> COMMIT
```

### Model

By default, the connection configuration will be read from `ActiveRecord::Base`.
If you want to use connection configuration from a different model, you can
can assign it to the database object after loading the extension:

```rb
class MyModel < ActiveRecord::Base
  connects_to database: { writing: :animals, reading: :animals_replica }
end
```
```rb
DB.activerecord_model = MyModel
```

## Tests

You'll first want to run the rake tasks for setting up databases and users:

```sh
$ rake db_setup_postgres
$ rake db_setup_mysql
```

Then you can run the tests:

```sh
$ rake test
```

When you're done, you can delete the created databases and users:

```sh
$ rake db_teardown_postgres
$ rake db_teardown_mysql
```

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in this project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/janko/sequel-activerecord-adapter/blob/master/CODE_OF_CONDUCT.md).

[Sequel]: https://github.com/jeremyevans/sequel
[Rodauth]: https://github.com/jeremyevans/rodauth
[rom-sql]: https://github.com/rom-rb/rom-sql
[sequel transaction hooks]: http://sequel.jeremyevans.net/rdoc/files/doc/transactions_rdoc.html#label-Transaction+Hooks
[Oracle enhanced]: https://github.com/rsim/oracle-enhanced
[SQL Server]: https://github.com/rails-sqlserver/activerecord-sqlserver-adapter
