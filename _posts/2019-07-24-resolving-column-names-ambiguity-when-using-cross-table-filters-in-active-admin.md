---
layout: post
title: Resolving Column Names Ambiguity When Using Cross Table Filters In ActiveAdmin
categories: [activeadmin, postgres]
author: gagan93
---

Recently, I was stuck solving a very unique issue while adding a new filter to an existing **[ActiveAdmin](https://github.com/activeadmin/activeadmin)** page. Consider two models:

```ruby
class User
columns
  id: Integer
  name: String
  email: String
  balance: Float
```

Here, `balance` is a field representing balance for a legacy wallet. Consider another model:

```ruby
class AnotherWallet
columns
  id: Integer
  user_id: Integer (belongs_to :user)
  phone_number: String
  balance: Float
```

I had an ActiveAdmin page for `AnotherWallet` with a filter on `user's email` and some scopes on `balance` column.

```ruby
ActiveAdmin.register AnotherWallet do
  menu parent: 'Wallets'

  filter :phone_number
  filter :user_email

  scope :active # where(active: true)
  scope :zero_balance # where(balance: 0)
end
```

The page worked fine before I added a filter on `user_email`. On filtering by `user_email`, I got the following error
```sql
PG::AmbiguousColumn: ERROR:  column reference "balance" is ambiguous
LINE 1: ... ("users"."email" ILIKE '%example%') AND (balance <=...
                                                             ^
SELECT COUNT(DISTINCT "another_wallets"."id") FROM "another_wallets"
  LEFT OUTER JOIN
"users" ON "users"."id" = "another_wallets"."user_id"
  WHERE
"users"."email" ILIKE '%example%'
  AND
balance = 0
```

The query was fired by the `zero_balance` scope to count how many wallets have zero balance. The reason for this error was self-explanatory - Postgres performed a join between tables which had same column name (`balance`), and it had no idea how to differentiate between both.

At SQL level, the fix is simple; We need to add the table name before specifying the column. But because _ActiveAdmin_ provides a DSL for creating everything, I thought there would be an option for this as well.

I spent hours debugging this, but the fix was very simple and was required at the model level. Rather than initializing scopes like this:

```ruby
class AnotherWallet < ApplicationRecord
  ...
  scope :zero_balance, { where(balance: 0 ) }
end
```

I had to mention the complete table name in scope

```ruby
class AnotherWallet < ApplicationRecord
  ...
  scope :zero_balance, -> { where('another_wallets.balance = 0') }
end
```

The resultant query was:

```sql
SELECT COUNT(DISTINCT "another_wallets"."id") FROM "another_wallets"
  LEFT OUTER JOIN
"users" ON "users"."id" = "another_wallets"."user_id"
  WHERE
"users"."email" ILIKE '%example%'
  AND
"another_wallets".balance = 0
```

So, that's all for now! Let us know if you have any other suggestions for the same.
