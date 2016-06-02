---
layout: post
title: "Fun With Time Zones in Rails"
date: 2016-06-01 14:45
comments: true
categories: [Ruby, Rails, Time Zones]
---

Recently, we had a need to limit when certain rake tasks would run. The tasks sync our data with a third party data store that doesn't change over night for our continental US-based customers. What we wanted is for the tasks to only run between 7am-1am Eastern (4am-10pm Western) which should cover typical waking hours. In other words, we want to ensure the tasks do not run between 1am-7am Eastern.

If you've ever worked with time zones before, then you know this is often trickier than you'd expect. This would be no different.

Thankfully, the Ruby [Time](http://ruby-doc.org/core-2.2.2/Time.html) and Rails [ActiveSupport::TimeWithZone](http://api.rubyonrails.org/classes/ActiveSupport/TimeWithZone.html) classes provide everything we need.

## Goal: Do not run certain rake tasks between 1am-7am Eastern

We want an easy way to skip running certain tasks during certain hours of the day. In addtion, we want to be able to define those hours of the day using the time zone of our choice so that it's easier to understand. Ideally, we'll be able to write a little helper class so we can do something like this in our rake tasks:

```ruby
namespace :mydatasync do

  desc 'Syncs the data for syncable things...'
  task sync: :environment do
    unless MyDataSync::RakeTaskTimeHelper.is_allowed_to_run
      # notify team that we're skipping this sync operation
      next
    end

    # sync stuff...
  end
end
```

<!-- more -->

## Finding the correct time zone

To achieve our goal, we first need find out how Rails refers to the Eastern time zone. To see all time zones in Rails, just run `rake time:zones:all`

```sh
> rake time:zones:all

...

* UTC -05:00 *
Bogota
Eastern Time (US & Canada)
Indiana (East)
Lima
Quito

...
```

Cool! For our purposes, we'll need to use **Eastern Time (US & Canada)** when dealing with time zones.


## Using our time zone in a UTC world

By default, Ruby and Rails default time to UTC. In Rails, this can be overridden in `config/application.rb`:

```ruby
# default time zone is UTC
config.time_zone = "Eastern Time (US & Canada)"
```

In practice, it's best to leave the system time zone set to UTC. Instead, we can use Rails **`Time.use_zone`** to override Ruby **`Time.zone`** inside a block. Once block execution completes, the original time zone is restored.

```ruby
# doing stuff in UTC...

Time.use_zone("Eastern Time (US & Canada)") do
  # do stuff in the Eastern time zone
  # date and time automatically translated from UTC to Eastern Time
end

# resume doing stuff in UTC...
```

## Putting it all together

Here's a small module we can use in our rake task to see if it is "allowed" or "not allowed" at the current time. Notice we check to see if the time is allowed in the **`Time.use_zone`** block.

```ruby
module MyDataSync
  module RakeTaskTimeHelper
    # default "not allowed" to 1am-7am US Eastern
    # a real implementation would use environment vars, not hard-coded values
    NOT_ALLOWED_START_HOUR = 1
    NOT_ALLOWED_END_HOUR = 7

    def self.is_allowed_to_run
      startHour = self.not_allowed_start_hour
      endHour = self.not_allowed_end_hour
      allowed = true

      Time.use_zone("Eastern Time (US & Canada)") do
        # we're not allowed from startHour up to, but not including, endHour
        allowed = false if (startHour...endHour).include?(Time.current.hour)
      end

      allowed
    end

    private
    def self.not_allowed_start_hour
      NOT_ALLOWED_START_HOUR
    end

    def self.not_allowed_end_hour
      NOT_ALLOWED_END_HOUR
    end
  end
end
```

## Testing our implementation

An implementation is nice, but a tested, working implementation is better. Once again we'll use **`Time.use_zone`**, but we'll also add [ActiveSupport::Testing::TimeHelpers](http://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html) **`travel_to`** to set the times we wish to test. The code inside the **`travel_to`** block will use the time we provide, then revert to the original time after block execution completes.

```
require 'test_helper'

class RakeTaskTimeHelperTest < ActiveSupport::TestCase

  test 'is_allowed_to_run returns true when time is outside the bounds' do
    allowedStartHour = 1
    allowedEndHour = 7

    # 0 up to, but not including, allowedStartHour
    (0...allowedStartHour).each {|hour|
      assert_time_allowed true, hour, 00, 00
    }

    # allowedEndHour up to, and including, 23
    (allowedEndHour..23).each {|hour|
      assert_time_allowed true, hour, 00, 00
    }
  end

  test 'is_allowed_to_run returns false when time is inside the bounds' do
    allowedStartHour = 1
    allowedEndHour = 7

    # allowedStartHour up to, but not including, allowedEndHour
    (allowedStartHour...allowedEndHour).each {|hour|
      assert_time_allowed false, hour, 42, 00
    }

  end

  def assert_time_allowed expectedAllowed, hours, minutes, seconds
    year = Time.current.year
    day = Time.current.day
    month = Time.current.month

    Time.use_zone("Eastern Time (US & Canada)") do
      # use Time.current.utc_offset to create new time in correct time zone
      travel_to Time.new(year, month, day, hours, minutes, seconds, Time.current.utc_offset) do
        # use UTC to simulate the production environment
        Time.use_zone("UTC") do
          assert_equal expectedAllowed, MyDataSync::RakeTaskTimeHelper.is_allowed_to_run
        end
      end
    end
  end
end
```

## Wrapping Up

Working with time zones is tricky. Many developers, myself included, have been tripped up by intricasies of working with time zones. However, by using Rails **`Time.use_zone`**, we were able to achieve our stated goal of writing simple code to short-circuit rake tasks when they're not allowed to run. And by using Rails test helper **`travel_to`** we were able to write tests to ensure our helper class implemenation is working correctly.

Finally, many thanks to [@datachomp](https://twitter.com/datachomp) and [@jagthedrummer](https://twitter.com/jagthedrummer) for helpful suggestions to improve this post.


----


## Resources

There is much more to time zones in Ruby and Rails than covered here. These blog posts were super helpful in understanding how to deal with time zones in Rails.

Elle Meredith's excellent two-part series: [It's About Time (Zones)](https://robots.thoughtbot.com/its-about-time-zones) and [A Case Study in Multiple Time Zones](https://robots.thoughtbot.com/a-case-study-in-multiple-time-zones).


And Nicklas Ramhö's wonderful [Working with time zones in Ruby on Rails](http://www.elabs.se/blog/36-working-with-time-zones-in-ruby-on-rails).
