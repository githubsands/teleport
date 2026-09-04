# Teleport

## Index

I. overview

II. authorization, authentication, and resource based access control

III. on client subscriptions for logs and other events

IV. job execution and output streaming

V. job states & lifecycle and their relation to server system internals

VI. client library and sdk 

## I. overview

There are currently start, status, stop

different api calls cross different internal services (2) and authorization boundaries (`job_submitter` & `watcher`).

### APIs & Internal Services:

(1) `Job`: exists to submit new jobs or cancel existing jobs from a `job_submitter`, and get job statuses - can only be used by both `job_submitter`

(2) `Subscriptions`: exists to receive events more specifically job log events from the server. events have `at most once delivery` delivery guarantees through streams. missed events can be replayed. can be used by `job_submitter` and `watcher`.

### Low level Components:

(1) `JobManager`: Registers new jobs and keeps track of job states for later status querying. Upon a new job request from grpc api, jobs are written to disk in the case the server 
falls down for durability guarantees

(2) `SubscriptionManager & EventStore`: Keeps track of all events for `watchers` and `job_submitters` 

(3) `Executor`: Keeps a queue of all jobs that are not running in buckets based of job execution time - jobs are pruned from the queue by a simple SJF (shortest job first) algorithm

If a jobs execution time is not given it will be placed in the longest bucket. 

If a jobs execution time does not match its actual execution time nothing happens for now but a logged warning. 

![Teleport architecture diagram](./architecture.png) // NOTE: this has not been updated and is a larger scoped project

## II. authorization, authentication, and resource based access control

The system has two different roles: `job_submitters` and `watchers`.

### Roles:

(1) `job_submitters`: clients who mTLS authenticate with `job_submitter` as their OA can use the both the `job` and `subscription` api

(2) `watcher`: clients who mTLS authenticate with `watcher` as their OA can use only `subscription` api


## III. on client subscriptions for logs and other events

both `job_submitter` and `watcher` can subscribe to logs.

logs events are delivered at an `at most once basis delivery guarantees` - no acknowldgement is required

once subscribed the server will push log based events to the clients.


## IV. `job api` and its relation to job object manipulation and status tracking

started jobs are submitted to the server and then committed for durability if a job is submitted
this does not necessarily mean it has been initiated as it could sit in the server's queue until
resources are avaliable

jobs can only be created if the user has authenticated with the server
already and created an account through the `account api`

stopped jobs are removed from the program - they cannot be restarted.  if a job is stopped 
an event is emitted in the event store to be streamed along side logs.

only the `job_submitter` is authorized to stop or receive job status.

## V. Job execution and output streaming

one job execution may contain many continual job outputs

a job's execution is streamed through LogEvents which is granularly broken down into `JobOutput` 
where each `JobOutput` can have an optional `ChunkMetaData`.

if a job output within a jobs execution window exceeds `MAX_BYTES_PER_LOG_EVENT` it is broken down into many frames across a incremented `frame_offset` range where the entire output is identified by a unique `chunk_id` within `ChunkMetaData`

one job output can be composed of many sequential chunks spanned among many `EventLogs`

the ordering of a job execution individual job outputs is determined by incrementally ordered `seq` within the `JobOutput`

logs events can be replayed based off a clients last seen LogEvent `seq` id


## VI. job states & lifecycle and their relation to server system internals

jobs can have the following states:

```
enum JobState {
  JOB_STATE_UNCOMMITTED = 1;
  JOB_STATE_COMMITTED = 1;
  JOB_STATE_PENDING = 1;
  JOB_STATE_RUNNING = 2;
  JOB_STATE_STOPPED = 4;
  JOB_STATE_COMPLETED = 5;
  JOB_STATE_FAILED = 6;
}
```

`uncommitted`: job has yet to be sent from the client or flushed to the servers disk through a WAL

`committed`: job has succesfully been flushed to the servers disk and is durable - if the server
            falls over prejob completion, stop, or failure it can be succesfully replayed upon server
            restart

`pending`: job is currently within its queue in its time designated bucket

`running`: job is running and subscribers can receive soft real time events if currently subscribed
         to the account that this job is owned to

`stopped`: job is stopped from running if job was in any state besides RUNNING an invalid request is 
           sent to who ever made the stop_job grpc request - job status is then updated in job manager

`completed`: job has succesfully completed on the server

`failed`: job has failed on the server

upon server fall over all `committed`, `pending`, or `running` workloads are replayed into the servers queue.
running workloads are currently not check pointed so if the server falls down they will have to be executed
from scratch.

## VII. client library and sdk 

client library provides two different sdks for `watcher` and `job_submitter`. 

both `watcher` and `job_submitter` can stream job output (or logs)

when a `job_submitter` submits a job SDK will generate a idempotent key for this workload 

TODO: implement idempotent key deduplication internally in the server

TODO: implement exponential backoffs in the client in the case the server is overloaded

### streaming

other nuances exist in terms of potential extremely large log event payloads - if a log payload is exceeds a 
configurable size it is split into chunks on the server side

the sdk is responsible for joining different chunks (or frames) spanned across multiplie events that belong to the same chunk to keep log messages meaningful

client sdk will implement deduplication - already seen `EventLogs` events will be stored in a hashset by `EventLog's` seq field as a key

TODO: implement TTL for each seen `EventLog` event

### authorization

`job_submitter` can use both the `subscription` and `job` api
`watcher` can only use the `subscription` api

