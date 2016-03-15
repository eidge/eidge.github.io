---
layout: post
title: Refactoring Rails models with Database Views
date: 2016-03-11
tag: markdown
blog: true
#star: true
---

For the past couple of weeks at [Nourish](http://nourishcare.co.uk) we've been
tackling shifts and rota planning.

Taking the chance to over simplify the problem, rotas are all about allocating
people to shifts on a given date. That means that in terms of data structures you
basically have a table for **shifts** that contains time (but not date) information
and **allocations** that belong to shifts and have date (but not time)
information.

# The data model

In terms of Rails models you end up with something like this:

```ruby
class Shift < ActiveRecord::Base
  has_many :allocations, dependent: :destroy

  # DB Fields:
  #
  # start  => String (format: 'HH:MM')
  # finish => String (format: 'HH:MM')
end

class Allocation < ActiveRecord::Base
  belongs_to :shift

  has_many :user_allocations, dependent: :destroy
  has_many :users, through: :user_allocations

  # DB Fields
  #
  # date => Date
end

class User < ActiveRecord::Base
  has_many :user_allocations, dependent: :destroy
  has_many :allocations, through: :user_allocations
end
```

Even though the data model is quite simple it presents a few challenges when we
actually start querying for data.

Imagine you want all allocations for the 1st of January active at 10:00 am:

```ruby
1st_jan = Date.parse('01-01-2016')
Allocation
  .joins(:shift)
  .where(date: 1st_jan)
  .where("shifts.start <= :time && :time < shifts.finish", time: '10:00')
```

**Simple enough right?** That's until a new requirement comes: We need to
support shifts that don't start and finish in the same day (imagine a night
shift starting at 22:00 and ending at 06:00 of the following day).

Data-wise that's easy to do. We remove the limitation where ```shift.start <= shift.finish```
and assume that if the finish time is before the start time then the shift
spawns midnight and roll's over to the next day.

While it is still possible to query this data using raw SQL and ActiveRecord, you'll start
accumulating big chunks of SQL everytime you have to join allocations on shifts
time.

To prove my point, it would look like something like this:

```ruby
1st_jan = Date.parse('01-01-2016')
Allocation
  .joins(:shift)
  .where(
    <<-SQL
      (
        allocations.date = :date
          AND shifts.start <= :time
          AND shifts.finish > :time && :time < shifts.finish
      )
      OR
      (
        allocations.date = :date
          AND shifts.start > shifts.finish
          AND shifts.start <= :time AND
          AND ...
      )
    SQL
    , # I won't even bother trying to write the rest here.
    date: 1st_jan,
    time: '10:00'
  )
```

This is error prone, unfriendly and just painful. And I haven't even tryed to
query intervals that fit into a certain shift allocation. This is just plain
wrong.

If only we had time intervals in the same table rather then one table with dates
and another one with time, if only...

# Views to the rescue

Database views let you simplify data access through virtual tables that can be
composed of any data you can query. Turns out they're a great way to simplify
complex queries and fortunately they're completly transparent to ActiveRecord as
they look just like a normal table (at least for reads, not writes).

Database views are not a shiny new thing. They've been around for decades but
for some reason Rails developers seem to have forgotten about them.

Bare with me and take a look at this patch of SQL:

```SQL
SELECT
  allocations.id,
  allocations.date,
  allocations.created_at,
  allocations.updated_at,
  allocations.shift_id,
  (allocations.date || ' ' || shifts.start)::timestamp as start_at,
  (
    CASE WHEN (shifts.start > shifts.finish) THEN
      (allocations.date || ' ' || shifts.finish)::timestamp + interval '1' day
    ELSE
      (allocations.date || ' ' || shifts.finish)::timestamp
    END
  ) as finish_at
FROM
  shift_allocations
INNER JOIN shifts ON shifts.id = shift_allocations.shift_id
```

This select statement will compose a new database view that we'll call
AllocationsWithTime. As you can see, instead of leaking business logic of
rolling the day over when the shift start is greater than the shift finish time
we encapsulate that in a view.

To actually instantiate your View take a look at [scenic](https://github.com/thoughtbot/scenic)
from thoughbot, it as simple as creating a migration:

```ruby
class CreateAllocationsWithTime < ActiveRecord::Migration
  def change
    create_view :allocations_with_time
  end
end
```

and your model would look like:

```ruby
class AllocationWithTime < ActiveRecord::Base
  belongs_to :shift

  scope :for, -> (datetime) do
    where(
      <<-SQL
        allocations_with_time.start_at <= :datetime
          AND allocations_with_time.finish_at > :datetime
      SQL
      ,
      datetime: datetime
    )
  end

  private

  def read_only?
    # Prevent trying to write to the view as that
    # would result in a postgres error anyway.
    true
  end
end
```

The best part about this is that we can now use ActiveRecord to query our
allocations in a simple way. Because we have datetime's instead of split date
and time information, interval query is simple and we don't need to worry about
the midnight crossing:

```ruby
1st_of_jan_at_10_am = Time.zone.parse('01/01/2016 10:00')
AllocationWithTime.for(1st_of_jan_at_10_am)
```

No SQL trains, no repeated business logic across queries just simple
ActiveRecord queries.

Sometimes all you need to simplify your SQL is a bit of SQL. If you enjoyed this
and want to get in touch give me a shout at [@eidgeare](https://twitter.com/eidgeare).
