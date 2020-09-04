---
layout: post
title:  Composing SQL queries with Arel
description: Blog post about how to make SQL queries more readable by using Arel
tags:
- ruby
- rails
- arel
- sql
---

I believe you agree that SQL queries as a long strings are not very readable. Fortunately, in the Ruby on Rails world there is an Arel. For context, let's say we have the following models:

```ruby
class Resource < ActiveRecord::Base
  belongs_to :environment
  belongs_to :permission
end

class Environment < ActiveRecord::Base
  has_many :resources
  has_and_belongs_to_many :permissions
end

class Permission < ActiveRecord::Base
  has_many :resources
  has_and_belongs_to_many :environments
end
```

As we see, the `Resource` belongs to the `Environment` and to the `Permission`. There is also a many-to-many relationship between `Permission` and `Environment`. The `Permission` defines which environments are allowed. So, the `Resource` is only available when it's `Environment` is assigned to it's `Permission`.

```ruby
class Resource < ActiveRecord::Base
  #  (...)
  def available?
    permission.environment_ids.include?(environment_id)
  end
end
```

Our task is to get a list of all available resources. Arel comes in handy here.

```ruby
Resource.joins(permission: :environments).where(Environment.arel_table[:id].eq(Resource.arel_table[:environment_id]))
```

By using `joins` we got all the resources that have at least one allowed environment. Next, we had to filter these resources where the `environment_id` matches the `id` of the allowed environment. This gives us the following SQL query:

```sql
SELECT "resources".* FROM "resources" INNER JOIN "permissions" ON "permissions"."id" = "resources"."permission_id" INNER JOIN "environments_permissions" ON "environments_permissions"."permission_id" = "permissions"."id" INNER JOIN "environments" ON "environments"."id" = "environments_permissions"."environment_id" WHERE "environments"."id" = "resources"."environment_id"
```

Arel version looks simpler, right? You can use [Scuttle](http://www.scuttle.io/) to translate your SQL to Arel.
