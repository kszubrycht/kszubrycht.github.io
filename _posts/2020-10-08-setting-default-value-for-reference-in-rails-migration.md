---
layout: post
title: Setting default value for reference in Rails migration
description: A blog post on how to write a Rails migration that adds a new reference with a default value
tags:
- ruby
- rails
- postgres
---

For context, let's create a new `User` model and add some user to the database.

```bash
$ bundle exec rails generate model User name:string
$ bundle exec rails db:migrate
$ echo 'User.create(name: "Joe")' | bundle exec rails console
```

Our task is to introduce a new `Group` model. Each `User` is associated with one `Group` and each `Group` is associated with many `Users`.

```bash
$ bundle exec rails generate model Group name:string
$ bundle exec rails generate migration AddGroupRefToUsers group:references
```

Ready! It's time to run migrations.

```bash
$ bundle exec rails db:migrate
== 20201007213907 CreateGroups: migrating =====================================
-- create_table(:groups)
   -> 0.0291s
== 20201007213907 CreateGroups: migrated (0.0293s) ============================

== 20201007213916 AddGroupRefToUsers: migrating ===============================
-- add_reference(:users, :group, {:null=>false, :foreign_key=>true})
rails aborted!
StandardError: An error has occurred, this and all later migrations canceled:

PG::NotNullViolation: ERROR:  column "group_id" contains null values
/Users/kamilszubrycht/Documents/Development/my_super_app/db/migrate/20201007213916_add_group_ref_to_users.rb:3:in `change'
bin/rails:4:in `<main>'

Caused by:
ActiveRecord::NotNullViolation: PG::NotNullViolation: ERROR:  column "group_id" contains null values
/Users/kamilszubrycht/Documents/Development/my_super_app/db/migrate/20201007213916_add_group_ref_to_users.rb:3:in `change'
bin/rails:4:in `<main>'

Caused by:
PG::NotNullViolation: ERROR:  column "group_id" contains null values
/Users/kamilszubrycht/Documents/Development/my_super_app/db/migrate/20201007213916_add_group_ref_to_users.rb:3:in `change'
bin/rails:4:in `<main>'
Tasks: TOP => db:migrate
(See full trace by running task with --trace)
```

Oops. That's where the problem begins. We are mainly interested in second migration which adds `group` reference to the `users` table.

```ruby
class AddGroupRefToUsers < ActiveRecord::Migration[6.0]
  def change
    add_reference :users, :group, null: false, foreign_key: true
  end
end
```

We can't add a `group` reference to the `users` table, because `users` table is not empty and `group_id` column is not allowed to be `null`. So, we need to make sure that every user is associated to the group. For this we have to create a new group and use its `id` as a defult value for the `group_id` column. Once reference is added we are free to change the default `group_id` value to `null`.

```ruby
class AddGroupRefToUsers < ActiveRecord::Migration[6.0]
  def up
    default_group_id = Class.new(ApplicationRecord)
                            .tap { |c| c.table_name = :groups }
                            .find_or_create_by(name: 'Default group')
                            .id
    add_reference :users, :group, null: false, foreign_key: true, default: default_group_id
    change_column_default :users, :group_id, nil
  end

  def down
    remove_reference :users, :group
  end
end
```

It's worth to mention that instead of using `Group` model, I created a new anonymous class which inherits from the `ApplicationRecord` and then I explicitly set the table name to `groups`. In general, using the model classes in migrations is considered to be an anti-pattern. Let's run migrations.

```bash
$ bundle exec rails db:migrate
== 20201007213916 AddGroupRefToUsers: migrating ===============================
-- add_reference(:users, :group, {:null=>false, :foreign_key=>true, :default=>1})
   -> 0.0147s
-- change_column_default(:users, :group_id, nil)
   -> 0.0045s
== 20201007213916 AddGroupRefToUsers: migrated (0.0432s) ======================
```

Success! \ (•◡•) /
