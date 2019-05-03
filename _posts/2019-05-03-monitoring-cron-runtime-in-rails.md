---
layout: post
title: Monitoring Cron Runtime In Ruby
categories: [ruby, whenever, redis, rake, active_admin]
author: gagan93
---

We have our web backend in Ruby on rails, and we use many gems for different purposes. One such useful gem is **[whenever](https://github.com/javan/whenever)**. We have hundreds of cronjobs running per day. As our system scaled, it was essential to make sure that cronjobs run in the expected time. For, e.g., if a cronjob has to run every hour, we must make sure that one copy must run in less than an hour so that there are no cases when multiple copies of the same cron are running. Though whenever provided a very clean syntax for defining and deploying cronjobs, there was no insight on what’s going on. So we started building a simple in-house solution.


## What we wanted to build
1. A method to calculate runtime for any cronjob.
2. A syntax in which we can define expected runtime for a job (which should be less than the running frequency of the job).
3. A datastore to keep the last runtime for any job.
4. A notification mechanism (Slack/ Hipchat/ Email).
5. A user interface to view the last runtime vs. expected runtime for all cronjobs.

Let’s discuss all these one by one and build a solution :D


## Calculating cron runtime
[whenever](https://github.com/javan/whenever) provides two ways of running rails scripts: rake tasks and runners. One easy way to calculate runtimes is by using rake hooks. Rake provides start and end hooks which we can use to check the start time and end time of a cron job. So first, we moved all whenever crons to rake based scripts (we had few jobs which were written using runner syntax). However, these hooks run on all rake tasks (some of which may not be cron jobs), so we namespaced cron jobs properly to make this hook run for cron based rake tasks only. Also, we used Redis hashes to store start_time, end_time and task details. Here is the sample code,

```ruby
# Rakefile
tasks = Rake.application.tasks
tasks.each do |task|
  next unless task.to_s.starts_with?('loconav_crons:')

  task.enhance ['logger:before']
  task.enhance do
    Rake::Task['logger:after'].invoke
  end
end

# lib/tasks/before_logger
namespace :logger do
  desc 'Perform some steps before a rake task executes'
  task before: :environment do
    task_name = ARGV[0]
    task_comment = Rake.application.tasks.select{ |task| task.to_s == task_name }.first&.comment
    CronMonitor.signal_cron_start(task_name, { start_time: Time.current, description: task_comment }) # stores in redis
    puts "#{Time.current} Starting task : #{task_name}"
  end
end

# after_logger
namespace :logger do
  desc 'Perform some steps after a rake task executes'
  task after: :environment do
    task_name = ARGV[0]
    end_time = Time.current
    Heartbeat.signal_cron_completion(task_name, { end_time: end_time }) # stores in redis
    details = JSON.parse(Heartbeat.cron_last_run_details(task_name))
    cron_runtime = end_time - details['start_time'].to_time
    task_details = ::Cron::ExecutionConstraints::SCHEDULES[task_name.to_sym] # Discussing about this soon
    runtime_threshold = task_details[:runtime_threshold]

    if cron_runtime > runtime_threshold
      msg  = "Task #{task_name} exceeded runtime.\nExpected : #{format_runtime(runtime_threshold)}"
      msg += "\nCurrent Runtime : #{format_runtime(cron_runtime)}"
      msg += "\nFrequency #{to_hh_mm(task_details[:frequency])}"
      msg += "\nAt : #{task_details[:run_at_ist]}" if task_details[:run_at_ist].present?
      Notifier.send_message(msg) # Notify somewhere (eg email / slack)

      puts "Runtime #{cron_runtime} exceeded against expected runtime. Notified via webhook"
    end
    puts "#{Time.current} Completing task : #{task_name}"
  end

  def to_hh_mm(seconds)
    seconds = seconds.to_i
    "#{ (seconds/3600) }h : #{ (seconds%3600)/60 }m"
  end

  def format_runtime(seconds)
    if seconds > 60
      to_hh_mm(seconds)
    else
      seconds
    end
  end
end
```

The after_logger notes end_time, takes the previous runtime from Redis and calculates job runtime. If this comes out to be more than the threshold, it can send a notification.


## Syntax for defining cron runtime
We checked whenever documentation and found we can define these things for a cronjob:

1. Running frequency (Eg. 1.day)
2. Time (Eg. 1:00 am)
3. Log files (standard and error)
4. Rake command details

The only thing we wanted to add was _runtime_threshold_. So for each task, we defined some decent threshold initially and modified it later as per actual job runtimes. We came up with our own hash based syntax in a module called Cron::ExecutionConstraints. Here, we also defined our time in IST as all the customers in this cluster are in IST otherwise whenever accepts time as UTC.

```ruby
module Cron
  module ExecutionConstraints
    SCHEDULES = {
      'loconav:daily_tasks:stop_data_collection_for_expired_vehicles': {
        frequency: 1.day,
        runtime_threshold: 2.minutes,
        run_at_ist: '1:00 am',
        standard_logger: 'log/stop_data_collection_for_expired_vehicles.log',
        error_logger: 'log/stop_data_collection_for_expired_vehicles_err.log',
      },
      'loconav:daily_tasks:cache_expiry_date_in_vehicles': {
        frequency: 1.day,
        runtime_threshold: 10.minutes,
        run_at_ist: '2:30 am',
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

  if constraints[:run_at_ist]
    constraints[:run_at_ist] = [constraints[:run_at_ist]] unless constraints[:run_at_ist].is_a? Array
    run_at_utc = constraints[:run_at_ist].map{ |run_at_ist| (Time.parse(run_at_ist) - 19800).strftime('%H:%M %P') }
    options[:at] = run_at_utc
  end

  every constraints[:frequency], options  do
    rake schedule, output: { standard: constraints[:standard_logger], error: constraints[:error_logger] }
  end
end
```


## A datastore to keep last runtime for any job
We are using redis for this purpose, code for which is wrapped in a module called `HeartBeat`. There is no particular need to use Redis for storing stats, and you could use MongoDB or even a SQL database for this.


## A notification mechanism
`Notifier.send_message(msg)` in the code snippet is also kept as a black box. You can configure any notification mechanism in this class.


## A user interface to view all this
We use [active_admin](https://github.com/activeadmin/activeadmin) as our admin platform. So, to build a view over this, we used active admin [custom pages](https://activeadmin.info/10-custom-pages.html). We built a page which iterates over the redis hash (that stores cron runtimes and other meta) and shows up last runtime details for each cron. It highlights a row in red if the job runtime is more than the expected runtime. Our view looks like this:

Inline-style:
![alt text](https://github.com/loconav-tech/blog/blob/master/images/blog_2/sample_view_active_admin.png?raw=true)

So, that's all for now! Let us know if you have any issues in implementing this for your application.
