# Why

You schedule recurring jobs with cron, but you're unhappy because your jobs lack **visibility**.

* Which jobs are currently running?
* Have any jobs failed recently?
* What was the output last time this job ran?
* How long does it take to run each job?

# Assumptions

* You already have periodic jobs running in production.
* The jobs are heterogenous: various platforms and languages.
* You don't want to waste time modifying applications.
* You don't want to use a SaaS solution for this.

# Features

A RESTful API.

* recording job start, updates, and completion
* querying active and historic job results

A tiny wrapper command that enables reporting on any existing cron job.

    30 4 * * * /opt/morning_backup.sh
    30 4 * * * trustee exec -h myserver /opt/morning_backup.sh

The wrapper `trustee exec` would send a message to `myserver` when the job is first started, every so often (heartbeat) while the job is running, and again finally when the job finishes. The heartbeat could include stdout/stderr, and the final message would include the exit code. Probably would want to support setting options in /etc/trustee and ~/.trustee instead of needing to specify them in the crontab. Another maybe useful flag would be TTL so the Trustee server can detect subsequent failures.

In some cases `trustee exec` might not be quite what you want. Maybe you have a complicated cron task and you want to profile the first part, second part, and total runtime. In that case, you could use `trustee begin` and `trustee end`. Or maybe this would be better addressed with something like `trustee split` like a stopwatch lap counter.

# Future Plans

* Receive over SQS/AMQP. *For jobs with many updates.*
* Output to Graphite and Logstash *(and maybe AMQP?)*
* Failure notifications by email or SNS. *Maybe as a separate component?*
* A beautiful frontend. *Definitely a separate installable component, but packaged together.*
* A report generator. *Ugh.*

# Should I also build a scheduler?

* Locking with Redis to ensure run-once semantics. *Maybe some people can get away without this because they're using a fancy container orchestrator or whatever. Or maybe they can use Heartbeat or Consul Lock to ensure that some cron-type process is running in only one place. Even Sensu could be used in [round robin](https://sensuapp.org/docs/latest/clients#round-robin-client-subscriptions) mode to run a short-lived but frequent scheduler process.*
* Schedule recurring HTTP callbacks
    * Synchronous: wait for 200 status
    * Async: provide a webhook URI callback to indicate completion
    * Static schedules in a config file
    * Programmatic creation of schedules with RESTful API *(so that schedule definitions can live inside the application doing the work)*
    * Renewal of dynamic schedules upon successful completion
    * Purging of dynamic schedules if TTL elapses without any successes
    * User-defined retry policy
        - Only retry within target window (I like the terminology used by [Condor](http://research.cs.wisc.edu/htcondor/manual/v7.6/2_12Time_Scheduling.html#SECTION003121200000000000000))
        - Retry on failure
        - Retry on partial completion
* API could provide list of upcoming jobs / estimated next run time.
* Send email to start job (to be completed by a human)
* Some way to invoke traditional processes? *(A tiny HTTP listener that maps URIs onto local processes? Or a process that loads a local config file and self-registers for callbacks?)* Example: letting curator delete old Elasticsearch indices.
* Avoid overlapping jobs with job capacity zones

## Scheduler Implementation Thoughts

An incremental scheduler is a short-lived process that looks for schedules that are ready to run, triggers their payloads, then exits. The main advantage is moving lock complexity outside the application and simplifying high availability. The main disadvantages are extra overhead in starting processes, reduced timing granularity, and worse throughput compared to a long-lived process. Maybe this could be balanced by running for a short but predictable duration.

* Incremental scheduler built on [Later.js](http://bunkat.github.io/later/)
* Distributed locking with [cronlock](https://github.com/kvz/cronlock)
* Maybe Postgres as the database

# Out of scope

* Process supervision. Use something like Runit or supervisor.
* Worker nodes. Plenty of other tools for that.
* Task graphs. This isn't a tool for workflow automation.
* Prioritization. No job should get left behind.

# Development Goals

* Be an assembly of highly composable parts
* Be a [twelve-factor app](http://12factor.net/)

# Related theory

* [Reliable Cron across the Planet](https://queue.acm.org/detail.cfm?id=2745840) describes how Google does cron

# Related software

* [fcron](http://fcron.free.fr/)
* [rcron](https://code.google.com/p/rcron/)
* [Agenda](https://github.com/rschmukler/agenda)
* [Later](http://bunkat.github.io/later/)
* [whenever](https://github.com/javan/whenever)
* [Celery](http://www.celeryproject.org/)
* [Clockwork](https://github.com/tomykaira/clockwork)
* [Jenkins](https://jenkins-ci.org/)
* [TeamCity](https://www.jetbrains.com/teamcity/)
* [Chronos](https://mesos.github.io/chronos/)
* [Quartz](http://quartz-scheduler.org/)
* [cronlock](https://github.com/kvz/cronlock)
* [cron-as-a-service](https://www.npmjs.com/package/cron-as-a-service)
* [norc](https://github.com/darrellsilver/norc)
* [Schyntax](http://bret.codes/schyntax-part-1/)
* [resque-scheduler](https://github.com/resque/resque-scheduler)
* [Jobber](http://dshearer.github.io/jobber/)
* [highlander](https://github.com/chriscannon/highlander) or [flock](http://man7.org/linux/man-pages/man2/flock.2.html)

## SaaS alternatives

* [Temporize](http://temporize.net/)
* [Azure Task Scheduler](http://azure.microsoft.com/en-us/services/scheduler/)
* [IronWorker](http://www.iron.io/pricing/#worker)
* [Google App Engine Cron](https://cloud.google.com/appengine/features/#cron)
* [AWS Elastic Beanstalk Worker](https://medium.com/@joelennon/running-cron-jobs-on-amazon-web-services-aws-elastic-beanstalk-a41d91d1c571)
* [Cronitor](https://cronitor.io/)
* DEFUNCT [Proby](http://probyapp.com/)
* [Steward](https://steward.io/) - very similar! lightweight.
* [SetCronJob](https://www.setcronjob.com/)
* [WebTask.io](https://github.com/auth0/wt-cli/tree/master/sample-webtasks#cron)
* [Pushmon](http://www.pushmon.com/cms/faq) - like [TimerCheck.io](https://alestic.com/2015/07/timercheck-scheduled-events-monitoring/)
* [Datadog](https://github.com/DataDog/documentation/issues/18#issuecomment-37036248)
* [Cronblast](https://cronblast.com/)
* [Dead Man's Snitch](https://deadmanssnitch.com/)
* [CronAlarm](https://www.cronalarm.com/)
* [cronasaservice](https://www.cronasaservice.com/)

# Who cares?

> The problem with this approach is that a new task request can arrive while a worker has not finished running the previous task. Care should be taken to avoid workers stepping over each other.
http://dejanglozic.com/2014/07/21/node-js-apps-and-periodic-tasks/

> Scheduled HTTP requests as a service: is there such a thing? For example: "Perform this API call every x hours"
https://twitter.com/toolmantim/status/621505246074793984

> cron sucks, any replacements that actually work?
https://twitter.com/tjholowaychuk/status/305117527212699648

> I am in need of a REST API to schedule tasks, modify existing schedule, etc.
https://github.com/mher/flower/issues/448#issue-100956833

> How to record time taken by a cron to execute.
http://serverfault.com/q/316527/23300

And here's some people that really don't care.

* https://news.ycombinator.com/item?id=8307305

# Other names that were considered

Ravel. Cronit. Anacron. Mexicron. Old Faithful. Good old reliable Jake. Perfectionist. Hound. Seasonal. Recurrent. Periodic. Harmonic. Cyclist. Metronome. Rebeat. Beat. Rhythm. Harmony.
