---
layout: post
title: Monitoring Cron Runtime In Ruby
categories: [whenever, rake]
author: gagan93
---

We have our web backend in Ruby on rails, and we use many gems for different purposes. One such useful gem is **[whenever](https://github.com/javan/whenever)**. We have thousands of cronjobs running per day. As our system scaled, it was essential to make sure that cronjobs run in the expected time. For, e.g., if a cronjob has to run every hour, we must make sure that cron runs in less than an hour. To avoid multiple copies of cron, whenever providers a [solution](https://github.com/javan/whenever/wiki/Exclusive-cron-task-lock-with-flock) using linux file locks. Though whenever provided a very clean syntax for defining and deploying cronjobs, there was no insight on what is going on. So we started building a simple in-house solution.


## What we wanted to build
1. A method to calculate runtime for any cronjob.
2. A syntax in which we can define expected runtime for a job (which should be less than the running frequency of the job).
3. A datastore to keep the last runtime for any job.
4. A notification mechanism (Slack/ Hipchat/ Email).
5. A user interface to view the last runtime vs. expected runtime for all cronjobs.

Let’s discuss all these one by one and build a solution :D


## Calculating cron runtime
_whenever_ provides two ways of running rails scripts: rake tasks and runners. One easy way to calculate runtimes is by using rake hooks. Rake provides start and end hooks which we can use to check the start time and end time of a cron job. So first, we moved all _whenever_ crons to rake based scripts (we had few jobs which were written using runner syntax). However, these hooks run on all rake tasks (some of which may not be cron jobs), so we namespaced cron jobs properly to make this hook run for cron based rake tasks only. Also, we used Redis hashes to store start_time, end_time and task details.

Here is the sample code,

```ruby
# Rakefile
tasks = Rake.application.tasks
tasks.each do |task|
  next unless task.to_s.starts_with?('cron:')

  task.enhance ['logger:before']
  task.enhance do
    Rake::Task['logger:after'].invoke
  end
end

# lib/tasks/before_logger.rake
namespace :logger do
  desc 'Perform some steps before a rake task executes'
  task before: :environment do
    task_name = ARGV[0]
    task_comment = Rake.application.tasks.select{ |task| task.to_s == task_name }.first&.comment
    CronMonitor.signal_cron_start(task_name, { start_time: Time.current, description: task_comment }) # stores in redis
    puts "#{Time.current} Starting task : #{task_name}"
  end
end

# lib/tasks/after_logger.rake
namespace :logger do
  desc 'Perform some steps after a rake task executes'
  task after: :environment do
    task_name = ARGV[0]
    end_time = Time.current

    # Stores details in redis. Explained later.
    CronMonitor.signal_cron_completion(task_name, { end_time: end_time })
    details = JSON.parse(CronMonitor.cron_last_run_details(task_name))
    cron_runtime = end_time - details['start_time'].to_time

    # ExecutionConstraints is our handy syntax to define cron with it's expected runtime. Explained later.
    task_details = ::Cron::ExecutionConstraints::SCHEDULES[task_name.to_sym]
    runtime_threshold = task_details[:runtime_threshold]

    if cron_runtime > runtime_threshold
      msg  = "Task #{task_name} exceeded runtime.\nExpected runtime: #(runtime_threshold)}"
      msg += "\nCurrent Runtime: #{cron_runtime}"
      msg += "\nFrequency #{task_details[:frequency]}"
      msg += "\nAt : #{task_details[:run_at]}" if task_details[:run_at].present?

      # Notify somewhere (eg email, webhook, etc)
      Notifier.cron_threshold_exceeded(msg)

      puts "Runtime #{cron_runtime} exceeded against expected runtime. Notified via webhook"
    end
    puts "#{Time.current} Completing task : #{task_name}"
  end
end
```

The after_logger notes end_time, takes the previous runtime from Redis and calculates job runtime. If this comes out to be more than the threshold, it can send a notification.


## Syntax for defining cron runtime
We checked _whenever_ documentation and found we can define the following for a cronjob:

1. Running frequency (Eg. 1.day)
2. Time (Eg. 1:00 am)
3. Log files (standard and error)
4. Rake command details

The only thing we wanted to add was _runtime_threshold_. So for each task, we initially defined some decent threshold and modified it later as per actual job runtimes. We came up with our own hash based syntax in a module called .

```ruby
module Cron
  module ExecutionConstraints
    SCHEDULES = {
      'cron:daily_tasks:stop_data_collection_for_expired_vehicles': {
        frequency: 1.day,
        runtime_threshold: 2.minutes,
        run_at: '1:00 am',
        standard_logger: 'log/stop_data_collection_for_expired_vehicles.log',
        error_logger: 'log/stop_data_collection_for_expired_vehicles_err.log',
      },
      'cron:daily_tasks:cache_expiry_date_in_vehicles': {
        frequency: 1.day,
        runtime_threshold: 10.minutes,
        run_at: '2:30 am',
        standard_logger: 'log/cache_expiry_date_in_vehicles.log',
        error_logger: 'log/cache_expiry_date_in_vehicles_err.log',
      }
    end
  end
end
```

The _SCHEDULES_ hash defines all the parameters we need. So now, the `schedule.rb` file looks like:

```ruby
require './app/modules/cron/execution_constraints.rb'

::Cron::ExecutionConstraints::SCHEDULES.each do |schedule, constraints|
  options = {}
  options[:at] = constraints[:run_at] if constraints[:run_at]

  every constraints[:frequency], options  do
    rake schedule, output: { standard: constraints[:standard_logger], error: constraints[:error_logger] }
  end
end
```


## A datastore to keep last runtime for a job
We are using redis as a datastore here, code for which is wrapped in a module called `CronMonitor`. There is no particular need to use Redis here, and you can use MongoDB or a SQL database for storing stats.

Here's the sample code for redis based implementation:
```ruby
module CronMonitor
  def self.signal_cron_start(task, details)
    redis_client.hset(CRON_STATS_REDIS_KEY, task, details.to_json)
  end

  def self.signal_cron_completion(task, details)
    existing_details = JSON.parse(redis_client.hget(CRON_STATS_REDIS_KEY, task))
    redis_client.hset(CRON_STATS_REDIS_KEY, task, existing_details.merge(details).to_json)
  end
end
```


## A notification mechanism
`Notifier.cron_threshold_exceeded(msg)` in the code snippet is also kept as a black box. You can configure any notification mechanism in this class.

Here's the sample code

```ruby
class Notifier < ActionMailer::Base
  def cron_threshold_exceeded(msg)
    @msg = msg
    mail(from: FROM_EMAIL, to: 'developers@domain.com', subject: '[WARNING] Cron Runtime Exceeded !')
  end
end
```


## A user interface to view all this
We use [active_admin](https://github.com/activeadmin/activeadmin) as our admin platform. So, to build a view over this, we used active admin [custom pages](https://activeadmin.info/10-custom-pages.html). We built a page to view this data and highlight the jobs which take longer than expected.

Here's what we needed to build

![alt text](https://github.com/loconav-tech/blog/blob/master/images/blog_2/sample_view_active_admin.png?raw=true)

So, that's all for now! Let us know if you have any issues in implementing this for your application.
