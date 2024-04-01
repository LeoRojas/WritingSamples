# Best Practices for Avoiding Database Migration Pitfalls with Ruby on Rails

In this article, we will explore the concept of Ruby on Rails migrations, why they are important, and what best practices we need to follow when creating or modifying them. Migrations help keep our database schema healthy and are essential when a data migration process must be handled. 

## What is a migration?
Migrations are ruby files that enable a convenient way to alter your database schema. The migrations use a Ruby DSL, so there is no need to make the database changes writing SQL. This allows the project to be database-independent.

Each migration is a set of changes for the database and the schema; both will reflect those changes once the migration is executed. This is handled by Active Record, Ruby on Rails ORM (Object Relational Mapping). The schema starts off empty, and each migration adds or removes tables or columns, changes column types, sets columns' default values, adds indexes, etc. Depending on the migration content. Active Record knows what migrations have been executed and in which order thanks to the migrations file name structure, YYYYMMDDHHMMSS_create_products.rb, a UTC timestamp is included to ensure sequential date-based execution.

Migrations are stored as files in the db/migrate directory. Migrations also specify the Ruby on Rails version that was used to create them. We can see this in the next section, “Creating New Migrations” on the second code snippet.

## Creating New Migrations
Now that we know what a migration is and some of its benefits, it would be good to learn how to create one. We can create them using the following convenient command; this will create an empty migration with the specified name:

$ bin/rails generate migration AddPartNumberToProducts

class AddPartNumberToProducts < ActiveRecord::Migration[6.0]
  def change
  end
end

A convenient feature of migrations is that we can specify their name in the form "AddXXXToYYY" or "RemoveXXXFromYYY" followed by a list of column names and types. Then, a migration containing the appropriate add_column and remove_column statements will be created. This saves us time and avoids typos.

$ bin/rails generate migration AddPartNumberToProducts part_number:string

class AddPartNumberToProducts < ActiveRecord::Migration
  def change
    add_column :products, :part_number, :string
  end
end


## Running Migrations
The following command executes every migration's change or up method that has yet to be run. The order in which they are executed is determined by the timestamp on the migration file name.

rails db:migrate

## Rollback and Recovery
A common task is to rollback the last migration. For example, if you made a mistake in it and wish to correct it. Rather than tracking down the version number associated with the previous migration, you can run:

$ bin/rake db:rollback

This will rollback the latest migration, either by reverting the change method or by running the down method. If you need to undo several migrations, you can provide a STEP parameter:

$ bin/rake db:rollback STEP=3

This will revert the last three migrations.

The db:migrate:redo task is a shortcut for doing a rollback and then migrating back up again. As with the db:rollback task, you can use the STEP parameter if you need to go more than one version back, for example:

$ bin/rake db:migrate:redo STEP=3

Using db:migrate, we can achieve the same result that we get when we use rollback and redo. But they are convenient because you do not need to specify the version to migrate to explicitly.
You can get your database to a specific version in time using this command:

$ bin/rake db:migrate VERSION=20080906120000

This will execute the change method, ups and downs methods in the migration files until the database schema reaches the version parameter, which is the time timestamp specified in the migrations file name.

## Executing SQL
Raw sql can be executed in a migration file. Let’s look at the following example.

class ExampleMigration < ActiveRecord::Migration
  def change
    create_table :products do |t|
      t.references :category
    end
 
    reversible do |dir|
      dir.up do
        #add a foreign key
        execute <<-SQL
          ALTER TABLE products
            ADD CONSTRAINT fk_products_categories
            FOREIGN KEY (category_id)
            REFERENCES categories(id)
        SQL
      end
      dir.down do
        execute <<-SQL
          ALTER TABLE products
            DROP FOREIGN KEY fk_products_categories
        SQL
      end
    end
 
    add_column :users, :home_page_url, :string
    rename_column :users, :email, :email_address
  end
end

We can see the command execute, which allows raw SQL to be executed inside a migration file. Raw SQL should always be the last option; most times, active record models or helpers are enough to achieve what we need.


## Why are migrations important?


Rails migrations arе a fundamеntal part of managing your databasе structure. Thеy offеr numеrous advantagеs, including:

### Rollback capability
The migrations are easy to rollback, so any error that raises due to a database schema problem can be solved fast using the commands that we presented earlier, like bin/rake db:rollback.

### Database Independence
Migrations can be used with any database engine/adapter that Active Record supports, this is thanks to the fact that the migrations are written in Ruby DSL and not SQL. This makes it easier to change the project's database engine.

### Schema fallback
When the schema gets outdated or gets “corrupted” for any reason, it can be deleted and rebuilt from scratch, executing all the migrations in the project codebase. This helps us to not rely completely on the database schema file. Is important to note that in the production environment, usually, the database schema is used when building the project, and not the migrations, because executing all migration takes more time than updating or building a database directly from the schema file.

### Portability or containerization
When all migrations are correctly written and can be executed without errors, it means that we can rebuild our project database anytime and as many times as we want; the database schema will be the same. When we try containerization with tools like docker, the database generation will be simple and less error-prone. 



## Best practices for migrations

### Avoid Irreversible Migrations

Irreversible Migrations are the ones that are not allowed to roll back the changes that are performed on the migration.
It’s best to avoid using irreversible migrations; their usage can lead to hard-to-fix issues.

It’s a good idea to ensure your migration can be undone by running rails db: rollback.

When making irreversible changes to the database, it’s a good practice to create a database backup before proceeding.

# Drop a column with caution
class RemoveColumnFromProducts < ActiveRecord::Migration[6.0]
  def change
    remove_column :products, :deprecated_column
  end


  def down
    raise ActiveRecord::IrreversibleMigration
  end
end

### Test Migrations in a Controlled Environment
Create test environments that mirror your production environment closely. Running migrations in these environments helps identify and address potential issues before reaching the production database.


# Run migrations in the test environment
$ rake db:migrate RAILS_ENV=test

### Implement Atomic Migrations
Keep each migration atomic, representing a single change to the database schema. Avoid combining multiple changes into a single migration, as this can complicate rollback procedures.

# Example of an atomic migration
class AddColumnToUsers < ActiveRecord::Migration[6.0]
  def change
    add_column :users, :new_column, :string
  end
end


### Avoid creating records inside migration files.
If you need to add data inside a migration file, make sure it’s the only way to achieve your goal. Most of the time, adding records in migration files can be replaced by rake tasks created for that specific purpose. There are similar cases in which updates to records need to be performed; for example, after adding a deleted boolean column to a model with a default value of false, we would need to update the existing records and set the new column's default value. This update can be done inside a migration file but also using a rake task; the choice is up to you.

### Prefer ActiveRecord Methods Over Raw SQL
Whenever possible, use ActiveRecord methods instead of raw SQL. This ensures better compatibility across different database systems supported by Rails.

### Keep Migrations Simple
Maintain simplicity in your migrations. If a migration becomes too complex, consider breaking it into multiple smaller migrations for easier maintenance. For example, if you want to add multiple columns to a table, it's better to split that migration into smaller/simpler migrations, one per column that you want to add.

### Monitoring

It’s always important to monitor our project's performance, logs, queries, and all kinds of metrics that we can get. This allows us to ensure that the project is not only “working” but using the hardware and resources we appropriately allocated for its operation. Constant monitoring is also crucial to quickly detect and fix any error that arises. Most monitoring tools or solutions available provide features and functionalities in a way that the performance of our application is no longer a black box, it can be checked on a regular basis and allows us to rest at ease. Some monitoring tools available are the following:

Dynatrace
Sentry
New Relic
Datadog
 
## Conclusion/Summary

Mastering database migrations in Ruby on Rails is crucial for maintaining a healthy and scalable application. By adhering to these best practices, you'll navigate through potential pitfalls, ensuring a smooth evolution of your database schema as your application grows. Remember, meticulous planning, testing, and version control are your allies in the journey of database migrations with Ruby on Rails.


