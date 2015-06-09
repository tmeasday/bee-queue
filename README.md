# Bee Queue [![NPM Version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Coverage Status][coveralls-image]][coveralls-url]

A simple, fast, robust job/task queue for Node.js, backed by Redis.
- Simple: ~500 LOC, and the only dependency is [node_redis](https://github.com/mranney/node_redis).
- Fast: maximizes throughput by minimizing Redis and network overhead.
- Robust: designed with concurrency, atomicity, and failure in mind; 100% code coverage.

```javascript
var Queue = require('bee-queue');
var queue = new Queue('example');

var job = queue.createJob({x: 2, y: 3}).save();
job.on('succeeded', function (result) {
  console.log('Received result for job ' + job.id + ': ' + result);
});

// Process jobs from as many servers or processes as you like
queue.process(function (job, done) {
  console.log('Processing job ' + job.id);
  return done(null, job.data.x + job.data.y);
});
```

## Overview
Bee-Queue is meant to power a distributed worker pool and was built with short, real-time jobs in mind. A web server can enqueue a job, wait for a worker to complete it, and return its results within an HTTP request. Scaling is as simple as running more workers.

[Celery](http://www.celeryproject.org/), [Resque](https://github.com/resque/resque), [Kue](https://github.com/LearnBoost/kue), and [Bull](https://github.com/OptimalBits/bull) operate similarly, but are generally designed for longer background jobs, supporting things like job scheduling and prioritization, which Bee-Queue [currently does not](#missing-features). Bee-Queue can handle longer background jobs just fine, but they aren't the primary focus.

- Create, save, and process jobs
- Concurrent processing
- Job timeouts and retries
- Pass events via Pub/Sub
  - Progress reporting
  - Send back job results
- Robust design
  - Strives for all atomic operations
  - Retries [stuck jobs](#under-the-hood)
- Performance-focused
  - Keeps [Redis usage](#under-the-hood) to the bare minimum.
  - Uses [Lua scripting](http://redis.io/commands/EVAL) and [pipelining](http://redis.io/topics/pipelining) to minimize network overhead
  - Benchmarks (coming soon)
- 100% code coverage


## Installation
```
npm install bee-queue
```
You'll also need [Redis 2.8+](http://redis.io/topics/quickstart) running somewhere.

# Table of Contents
- [Motivation](#motivation)
- [Creating Queues](#creating-queues)
- [Creating Jobs](#creating-jobs)
- [Processing Jobs](#processing-jobs)
- [Progress Reporting](#progress-reporting)
- [Job Events](#job-events)
- [Queue Events](#queue-events)
- [Stalled Jobs](#stalled-jobs)
- [API Reference](#api-reference)
- [Under The Hood](#under-the-hood)
- [Contributing](#contributing)
- [License](https://github.com/LewisJEllis/bee-queue/blob/master/LICENSE)

## Motivation
Celery is for Python, and Resque is for Ruby, but [Kue](https://github.com/LearnBoost/kue) and [Bull](https://github.com/OptimalBits/bull) already exist for Node, and they're good at what they do, so why does Bee-Queue also need to exist?

In short: we needed to mix and match things that Kue does well with things that Bull does well, and we needed to squeeze out more performance. There's also a [long version](https://github.com/LewisJEllis/bee-queue/wiki/Origin) with more details.

Bee-Queue starts by combining Bull's simplicity and robustness with Kue's ability to send events back to job creators, then focuses heavily on minimizing overhead, and finishes by being strict about [code quality](https://github.com/LewisJEllis/bee-queue/blob/master/.eslintrc) and [testing](https://coveralls.io/r/LewisJEllis/bee-queue?branch=master). It compromises on breadth of features, so there are certainly cases where Kue or Bull might be preferable (see below).

Bull and Kue do things really well and deserve a lot of credit. Bee-Queue borrows ideas from both, and Bull was an especially invaluable reference during initial development.

#### Missing Features
- Job scheduling: Kue and Bull do this.
- Worker tracking: Kue does this.
- All-workers pause-resume: Bull does this.
- Web interface:  Kue has a nice one built in, and someone made [one for Bull](https://github.com/ShaneK/Matador).
- Job priority: multiple queues get the job done in simple cases, but Kue has first-class support. Bull provides a wrapper around multiple queues.

Some of these are potential future additions; please comment if you're interested in any of them!

#### Why Bees?
Bee-Queue is like a bee because it:
- is small and simple
- is fast (bees can fly 20mph!)
- carries pollen (messages) between flowers (servers)
- something something "worker bees"

## Creating Queues
[Queue](#queue) objects are the starting point to everything this library does. To make one, we just need to give it a name, typically indicating the sort of job it will process:
```javascript
var Queue = require('bee-queue');
var addQueue = new Queue('addition');
```
Queues are very lightweight - the only significant overhead is connecting to Redis - so if you need to handle different types of jobs, just instantiate a queue for each:
```javascript
var subQueue = new Queue('subtraction', {
  redis: {
    host: 'somewhereElse'
  },
  isWorker: false
});
```
Here, we pass a `settings` object to specify an alternate Redis host and to indicate that this queue will only add jobs (not process them). See [Queue Settings](#settings) for more options.

## Creating Jobs
Jobs are created using `Queue.createJob(data)`, which returns a [Job](#job) object storing arbitrary `data`.

Jobs have a chaining API with commands `.retries(n)` and `.timeout(ms)` for setting options, and `.save([cb])` to save the job into Redis and enqueue it for processing:

```javascript
var job = addQueue.createJob({x: 2, y: 3});
job.timeout(3000).retries(2).save(function (err, job) {
  // job enqueued, job.id populated
});
```

Jobs can later be retrieved from Redis using [Queue.getJob](#queuegetjobjobid-cberr-job), but most use cases won't need this, and can instead use [Job Events](#job-events) and [Queue Events](#queue-events).

## Processing Jobs
To start processing jobs, call `Queue.process` and provide a handler function:
```javascript
addQueue.process(function (job, done) {
  console.log('Processing job ' + job.id);
  return done(null, job.data.x + job.data.y);
});
```
The handler function is given the job it needs to process, including `job.data` from when the job was created. It should then pass results to the `done` callback. For more on handlers, see [Queue.process](queueprocessconcurrency-handlerjob-done).


`.process` can only be called once per Queue instance, but we can process on as many instances as we like, spanning multiple processes or servers, as long as they all connect to the same Redis instance. From this, we can easily make a worker pool of machines who all run the same code and spend their lives processing our jobs, no matter where those jobs are created.

`.process` can also take a concurrency parameter. If your jobs spend most of their time just waiting on external resources, you might want each processor instance to handle 10 at a time:
```javascript
var baseUrl = 'http://www.google.com/search?q=';
subQueue.process(10, function (job, done) {
  http.get(baseUrl + job.data.x + '-' + job.data.y, function (res) {
    // parse the difference out of the response...
    return done(null, difference);
  });
});
```

## Progress Reporting
Handlers can send progress reports, which will be received as events on the original job instance:
```javascript
var job = addQueue.createJob({x: 2, y: 3}).save();
job.on('progress', function (progress) {
  console.log('Job ' + job.id + ' reported progress: ' + progress + '%');
});

addQueue.process(function (job, done) {
  // do some work
  job.reportProgress(30);
  // do more work
  job.reportProgress(80);
  // do the rest
  done();
});
```
Just like `.process`, these `progress` events work across multiple processes or servers; the job instance will receive the progress event no matter where processing happens.

## Job Events
Progress reporting happens via 'Job events'. Jobs also emit `succeeded`, `failed`, and `retrying` events:

## Queue Events

## Stalling Jobs

# API Reference

## Queue

### Settings
The default Queue settings are:
```javascript
var queue = Queue('test', {
  prefix: 'bq',
  stallInterval: 5000,
  redis: {
    host: '127.0.0.1',
    port: 6379,
    db: 0,
    options: {}
  },
  getEvents: true,
  isWorker: true,
  sendEvents: true,
  removeOnSuccess: false,
  catchExceptions: false
});
```
The `settings` fields are:
- `prefix`: string, default `bq`. Useful if the `bq:` namespace is, for whatever reason, unavailable on your redis database.
- `stallInterval`: number, ms; the length of the window in which workers must report that they aren't stalling. Higher values will reduce Redis/network overhead, but if a worker stalls, it will take longer before its stalled job(s) will be retried.
- `redis`: object, specifies how to connect to Redis.
  - `host`: string, Redis host.
  - `port`: number, Redis port.
  - `socket`: string, Redis socket to be used instead of a host and port.
  - `db`: number, Redis [DB index](http://redis.io/commands/SELECT).
  - `options`: options object, passed to [node_redis](https://github.com/mranney/node_redis#rediscreateclient).
- `isWorker`: boolean, default true. Disable if this queue will not process jobs.
- `getEvents`: boolean, default true. Disable if this queue does not need to receive job events.
- `sendEvents`: boolean, default true. Disable if this worker does not need to send job events back to other queues.
- `removeOnSuccess`: boolean, default false. Enable to keep memory usage down by automatically removing jobs from Redis when they succeed.
- `catchExceptions`: boolean, default false. Only enable if you want exceptions thrown by the [handler](#queueprocessconcurrency-handlerjob-done) to be caught by Bee-Queue and interpreted as job failures. Communicating failures via `done(err)` is preferred.

### Properties
- `name`: string, the name passed to the constructor.
- `keyPrefix`: string, the prefix used for all Redis keys associated with this queue.
- `paused`: boolean, whether the queue instance is paused.
- `settings`: object; the settings determined between those passed and the defaults

### Local Events

#### ready
```javascript
queue.on('ready', function () {
  console.log('queue now ready to start doing things');
});
```
The queue has connected to Redis and ensured that Lua scripts are cached.

#### error
```javascript
queue.on('error', function (err) {
  console.log('A queue error happened: ' + err.message);
});
```
Any Redis errors are re-emitted from the Queue.

#### succeeded
```javascript
queue.on('succeeded', function (job, result) {
  console.log('Job ' + job.id + ' succeeded with result: ' + result);
});
```
This queue has successfully processed `job`. If `result` is defined, the handler called `done(null, result)`.

#### retrying
```javascript
queue.on('retrying', function (job, err) {
  console.log('Job ' + job.id + ' failed with error ' + err.message ' but is being retried!');
});
```
This queue has processed `job`, but it reported a failure and has been re-enqueued for another attempt. `job.options.retries` has been decremented accordingly.

#### failed
```javascript
queue.on('failed', function (job, err) {
  console.log('Job ' + job.id + ' failed with error ' + err.message);
});
```
This queue has processed `job`, but its handler reported a failure with `done(err)`.

### Pub/Sub Events
These events are all reported by some worker queue (with `sendEvents` enabled) and sent as Redis Pub/Sub messages back to any queues listening for them (with `getEvents` enabled). This means that listening for these events is effectively a monitor for all activity by all workers on the queue.

If the `jobId` of an event is for a job that was created by that queue instance, a corresponding [job event](#job-events) will be emitted from that job object.

Note that Queue-level PubSub events pass the `jobId`, but do not have a reference to the job object, since that job might have originally been created by some other queue in some other process. [Job-level events](#job-events) are emitted only in the process that created the job, and are emitted from the job object itself.

#### job succeeded
```javascript
queue.on('job succeeded', function (jobId, result) {
  console.log('Job ' + job.id + ' succeeded with result: ' + result);
});
```
Some worker has successfully processed job `jobId`. If `result` is defined, the handler called `done(null, result)`.

#### job retrying
```javascript
queue.on('job retrying', function (jobId, err) {
  console.log('Job ' + jobId + ' failed with error ' + err.message ' but is being retried!');
});
```
Some worker has processed job `jobId`, but it reported a failure and has been re-enqueued for another attempt.

#### job failed
```javascript
queue.on('job failed', function (jobId, err) {
  console.log('Job ' + jobId + ' failed with error ' + err.message);
});
```
Some worker has processed `job`, but its handler reported a failure with `done(err)`.

#### job progress
```javascript
queue.on('job progress', function (jobId, progress) {
  console.log('Job ' + jobId + ' reported progress: ' + progress + '%');
});
```
Some worker is processing job `jobId`, and it sent a [progress report](#jobreportprogressn) of `progress` percent.

### Methods

#### Queue(name, [settings])

Used to instantiate a new queue; opens connections to Redis.

#### Queue.createJob(data)
```javascript
var job = queue.createJob({...});
```
Returns a new [Job object](#job) with the associated [user data](#job).

#### Queue.process([concurrency], handler(job, done))

Begins processing jobs with the provided handler function.

This function should only be called once, and should never be called on a queue where `isWorker` is false.

The optional `concurrency` parameter sets the maximum number of simultaneously active jobs for this processor. It defaults to 1.

The handler function should:
- Call `done` exactly once
  - Use `done(err)` to indicate job failure
  - Use `done()` or `done(null, result)` to indicate job success
    - `result` must be JSON-serializable (for `JSON.stringify`)
- Never throw an exception, unless `catchExceptions` has been enabled.
- Never ever [block](http://www.slideshare.net/async_io/practical-use-of-mongodb-for-nodejs/47) [the](http://blog.mixu.net/2011/02/01/understanding-the-node-js-event-loop/) [event](http://strongloop.com/strongblog/node-js-performance-event-loop-monitoring/) [loop](http://zef.me/blog/4561/node-js-and-the-case-of-the-blocked-event-loop) (for very long). If you do, the stall detection might think the job stalled, when it was really just blocking the event loop.

#### Queue.checkStalledJobs([cb])


#### Queue.close([cb])
Closes the queue's connections to Redis.

## Job

### Properties
- `id`: number, Job ID unique to each job. Not populated until `.save` calls back.
- `data`: object; user data associated with the job. It should:
  - Be JSON-serializable (for `JSON.stringify`)
  - Never be used to pass large pieces of data (100kB+)
  - Ideally be as small as possible (1kB or less)
- `options`: object used by Bee-Queue to store timeout, retries, etc.
  - Do not modify directly; use job methods instead.
- `queue`: the Queue responsible for this instance of the job. This will either:
  - the queue that called `createJob` to make the job
  - the queue that called `process` to process it
- `progress`: number; progress between 0 and 100, as reported by `reportProgress`.

### Job Events
These are all Pub/Sub events like [Queue PubSub events](#pubsub-events) and are disabled when `getEvents` is false.

#### succeeded
```javascript
var job = queue.createJob({...}).save();
job.on('succeeded', function (result) {
  console.log('Job ' + job.id + ' succeeded with result: ' + result);
});
```
The job has succeeded. If `result` is defined, the handler called `done(null, result)`.

#### retrying
```javascript
job.on('retrying', function (err) {
  console.log('Job ' + job.id + ' failed with error ' + err.message ' but is being retried!');
});
```
The job has failed, but it is being automatically re-enqueued for another attempt. `job.options.retries` has been decremented accordingly.

#### failed
```javascript
job.on('failed', function (err) {
  console.log('Job ' + job.id + ' failed with error ' + err.message);
});
```
The job has failed, and is not being retried.

#### progress
```javascript
job.on('progress', function (progress) {
  console.log('Job ' + job.id + ' reported progress: ' + progress + '%');
});
```
The job has sent a [progress report](#jobreportprogressn) of `progress` percent.

### Methods

#### Job.retries(n)
```javascript
var job = queue.createJob({...}).retries(3).save();
```
Sets how many times the job should be automatically retried in case of failure.

Stored in `job.options.retries` and decremented each time the job is retried.

Defaults to 0.

#### Job.timeout(ms)
```javascript
var job = queue.createJob({...}).timeout(10000).save();
```
Sets a job runtime timeout; if the job's handler function takes longer than the timeout to call `done`, the worker assumes the job has failed and reports it as such.

Defaults to no timeout.

#### Job.save([cb])
```javascript
var job = queue.createJob({...}).save(function (err, job) {
  console.log('Saved job ' + job.id);
});
```
Saves a job, queueing it up for processing. After the callback fires, `job.id` will be populated.

#### Job.reportProgress(n)
```javascript
queue.process(function (job, done) {
  ...
  job.reportProgress(10);
  ...
  job.reportProgress(50);
  ...
});
```
Reports job progress when called within a handler function. Causes a `progress` event to be emitted.

### Defaults

All methods with an optional callback field will use the following default:
```javascript
var defaultCb = function (err) {
  if (err) throw err;
};
```

Defaults for Queue `settings` live in `lib/defaults.js`. Changing that file will change Bee-Queue's default behavior.

## Under the hood

Each Queue uses the following Redis keys:
- `bq:name:id`: Integer, incremented to determine the next Job ID.
- `bq:name:jobs`: Hash from Job ID to a JSON string of its data and options.
- `bq:name:waiting`: List of IDs of jobs waiting to be processed.
- `bq:name:active`: List of IDs jobs currently being processed.
- `bq:name:succeeded`: Set of IDs of jobs which succeeded.
- `bq:name:failed`: Set of IDs of jobs which failed.
- `bq:name:stalling`: Set of IDs of jobs which haven't 'checked in' during this interval.
- `bq:name:events`: Pub/Sub channel for workers to send out job results.

Bee-Queue is non-polling, so idle workers are listening to receive jobs as soon as they're enqueued to Redis. This is powered by [brpoplpush](http://redis.io/commands/BRPOPLPUSH), which is used to move jobs from the waiting list to the active list. Bee-Queue generally follows the "Reliable Queue" pattern described on the [rpoplpush page](http://redis.io/commands/rpoplpush).

The `isWorker` [setting](#settings) creates an extra Redis connection dedicated to `brpoplpush`, while `getEvents` creates one dedicated to receiving Pub/Sub events. As such, these settings should be disabled if you don't need them; in most cases, only one of them needs to be enabled.

The stalling set is a snapshot of the active list from the beginning of the latest stall interval. During each stalling interval, workers remove their job IDs from the stalling set, so at the end of an interval, any jobs IDs left in the stalling set have missed their window (stalled) and need to be rerun. When `checkStalledJobs` runs, it re-enqueues any jobs left in the stalling set (to the waiting list), then takes a snapshot of the active list and stores it in the stalling set.

## Contributing
Pull requests are welcome; just make sure `grunt test` passes. For significant changes, open an issue for discussion first.

You'll need a local redis server to run the tests. Note that running them will delete any keys that start with `bq:test:`.

[npm-image]: https://img.shields.io/npm/v/bee-queue.svg?style=flat
[npm-url]: https://www.npmjs.com/package/bee-queue
[travis-image]: https://img.shields.io/travis/LewisJEllis/bee-queue.svg?style=flat
[travis-url]: https://travis-ci.org/LewisJEllis/bee-queue
[coveralls-image]: https://coveralls.io/repos/LewisJEllis/bee-queue/badge.svg?branch=master
[coveralls-url]: https://coveralls.io/r/LewisJEllis/bee-queue?branch=master
