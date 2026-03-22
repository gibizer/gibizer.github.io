---
title: Eventlet removal - Gazpacho status
date: 2026-03-14 18:45:00 +0200
categories: [OpenStack, Eventlet]
tags: [openstack, nova, eventlet, upstream]
hidden: true
---
Bah, it's been a long time. I published my last update 10 months ago. I'm sorry.
It was not the plan. But like any real plan, mine also died soon after meeting
the enemy. Time.

Due to the size of the time gap to cover, I won't try to make this a
chronological update, but rather a status report focusing on where we are at
the end of the Gazpacho release. Then, I will tell some selected stories from
the past one and a half cycles to show not just the result but a bit of the
journey as well.

Hang on, this will be a long one.

## Nova's Eventlet status at Gazpacho (2026.1) release

One can say that we are *almost* done. (I love famous last words.) If I look at
it from a certain angle and from a great distance then sure, we are *almost*
done. You can take the Gazpacho release, [configure your services to run with
native threading
](https://docs.openstack.org/nova/latest/admin/concurrency.html) and boom, no
more Eventlet in your nova deployment... *Almost*...

Our APIs (nova-api, nova-metadata)
[run with native threading by default now](https://review.opendev.org/c/openstack/nova/+/965924)
as well as the nova-scheduler. So you don't even need to enable native
threading mode there. But if you encounter issues then please report them to us
as a [bug](https://bugs.launchpad.net/nova/+filebug), and you can still
[switch back](https://docs.openstack.org/nova/latest/admin/concurrency.html#selecting-concurrency-mode-for-a-service)
to Eventlet mode while we are fixing those nasty bugs.

The nova-conductor and nova-compute still run with Eventlet by default as this
is our first release where these services gained the capability to run without
Eventlet. Still, you can follow our [guide](https://docs.openstack.org/nova/latest/admin/concurrency.html#selecting-concurrency-mode-for-a-service)
and switch them to native threading. Please do that in pre-production first.
Especially nova-compute can cause surprises, so we ask for caution and we are
eager for your feedback.

And last, our console proxies are only working with Eventlet at the time of
Gazpacho. There is [work](https://review.opendev.org/c/openstack/nova/+/976089)
underway to change that, but the release deadline hit before we could finish it.

We are also doing testing on multiple fronts. Our [nova-next CI job](https://github.com/openstack/nova/blob/7e0f18ff2baaf6aec89eb300b6942c488541da7e/.zuul.yaml#L428)
runs all services that support it in native threading mode and the tempest
results are green and stable. We also added a job,
[nova-alt-configuration](https://github.com/openstack/nova/blob/7e0f18ff2baaf6aec89eb300b6942c488541da7e/.zuul.yaml#L180)
that keeps testing all our services with the deprecated Eventlet support.
The rest of the CI jobs use each service's default mode during testing. This
includes the jobs run by other OpenStack projects. This means that as of
Gazpacho the greater OpenStack project already gates with native threaded
nova-api, nova-metadata, and nova-scheduler without apparent issues. We are
planning to change our default concurrency mode to native threading for
nova-conductor and nova-compute too early in the Hibiscus cycle.

Making our functional tests run with Eventlet is a task
[in progress](https://review.opendev.org/c/openstack/nova/+/979850). But the
majority of the unit tests are running in both modes in
[separate jobs](https://github.com/openstack/nova/blob/7e0f18ff2baaf6aec89eb300b6942c488541da7e/.zuul.yaml#L55).
But note that there is a short list of unit test
[excludes](https://github.com/openstack/nova/blob/7e0f18ff2baaf6aec89eb300b6942c488541da7e/threading_unit_test_excludes.txt)
that are still disabled in native threading mode. For sure we want this list
shrink to zero soon, but we are not there yet.

At the end of the Flamingo release we made nice
[progress](https://review.opendev.org/c/openstack/nova/+/960130) about scale
testing our APIs and the nova-scheduler in the upstream gate with Rally, but
this work was the one that needed to stop in Gazpacho to make room for the
work on nova-compute. Nevertheless we have some guidance on how to scale
these services in native threading mode in our [doc](https://docs.openstack.org/nova/latest/admin/concurrency.html#tunables-for-the-native-threading-mode).

And that is just the visible parts that are not done yet. During Gazpacho we
needed to make shortcuts and accumulated technical debt intentionally to make
it possible to release nova-compute with native support. These code crufts,
and hacks need to be cleaned up in the near future. Also as soon as we hit the
time to drop the support of the Eventlet mode we will have a series of
refactorings to remove all the complications currently needed to support both
modes.

### The numbers
In Gazpacho we landed 59 commits for Eventlet removal which shows that we
upped our game compared to Flamingo where this number was 39.

I tried to generate the bar chart of the number of remaining Eventlet mentions
in the codebase, but the numbers do not line up nicely. In general we have
more code lines now that refer to "eventlet" than when we started. In
reality however those mentions are a lot more centralized. We moved the
majority of our Eventlet logic to `nova.utils`, but that file now has a lot
more Eventlet specific lines. Also we need to test those utils, so we have a
bunch of Eventlet specific test cases, hence the overall numbers are way up.

Instead it makes sense to look at the files that are importing Eventlet:
```bash
$ egrep -l -R -e "import eventlet|from eventlet" ./nova
./nova/virt/libvirt/host.py
./nova/monkey_patch.py
./nova/utils.py
./nova/console/websocketproxy.py
./nova/tests/unit/virt/libvirt/test_host.py
./nova/tests/unit/virt/libvirt/test_driver.py
./nova/tests/unit/virt/libvirt/volume/test_mount.py
./nova/tests/unit/virt/vmwareapi/test_driver_api.py
./nova/tests/unit/virt/disk/mount/test_nbd.py
./nova/tests/unit/test_hacking.py
./nova/tests/unit/test_utils.py
./nova/tests/unit/storage/test_rbd.py
./nova/tests/fixtures/notifications.py
./nova/tests/fixtures/nova.py
```
* `nova.virt.libvirt.host` needed two distinct libvirt event handling
  implementations one for eventlet and one for native threading.
* `nova.monkey_patch` and `nova.utils` are our main code to handle selecting
  and using the proper concurrency primitives, so that the rest of the codebase
  does not need to care.
* `nova.console.websocketproxy` is our main remaining Eventlet only codepath to
  translate to native threading.
* `nova.test.fixtures` hides most of our unit tests from the concurrency mode
  they are running with. The notification fixture here is probably an oversight
  due to not having our functional tests running in both modes yet.
* `nova.tests.unit` holds all of our unit tests and some modules have
  significantly different behavior in the two concurrency modes, so there are
  mode specific test cases there depending still on Eventlet.

## The journey

### Adapting our timeline to reality

Based on the progress in Flamingo we raised the flag to the TC that the
Eventlet removal timeline of OpenStack needs to be revised to make it more
realistic for Nova. This went smoother than I anticipated and led to some
extra
[clarification](https://governance.openstack.org/tc//goals/selected/remove-eventlet.html#completion-criteria)
about the relationship between this work and our SLURP upgrade support.

### We need more Executors

For me the biggest learning came from realizing how dependent our codebase
is on different types of Executors. At first glance an
[Executor](https://docs.openstack.org/futurist/latest/reference/index.html#executors)
is just a good old thread pool. A bunch of worker threads waiting for new tasks
to execute while providing
[Future](https://docs.openstack.org/futurist/latest/reference/index.html#futurist.Future)
objects for the callers to gather the results. When we translated the APIs and
the scheduler we already used them happily as an abstraction over GreenThreads
and native Threads. It was nice and easy.

Sure we needed to be careful about the size of our Executors. The native
worker threads are a lot more expensive beasts than the GreenThreads. In the
past we had Executors defined to allow thousands of GreenThreads by default.
That would be very memory consuming in native threading. This led to some
tweaking of our default configuration in native threading mode. Make sure
you read the related release
[notes](https://github.com/openstack/nova/blob/master/releasenotes/notes/deprecate-unlimited-max_concurrent_live_migrations-29c54c7eeb77041c.yaml)
before you hit new limits.

But size is just an integer. (Another famous last words.) The nova-compute ups
the game with two very different needs regarding Executors:

#### Here is a task, but give me some time to decide if I really need it

At least the use case here is simple to explain. We have VMs, those VMs can
be hard rebooted via the API. We implement that by telling libvirt to reboot
the VM. Simple, you see. But.

We also want to detect when a VM stops itself, or
crashes. Libvirt happily sends us events about the VM power state changes. So
we listen to the STOPPED event and update the VM power state when we receive
it.

Now put the two behaviors together. During hard reboot the VM stops, libvirt
sends the STOPPED event, then the VM starts, libvirt sends the RESUMED event.
This event handling happens concurrently with nova's hard reboot codepath that
also sets the power state of the VM in the DB. It is easy to see how racy that
two concurrent power state updates can be. Also updating the DB twice during
a hard reboot is simply a waste of DB and message bus capacity. So for a long
time nova had the logic that when a libvirt STOPPED event arrived, it created
a power state update task, but did not immediately run it. Instead it told
Eventlet to run it after a couple seconds of delay. And if a RESUMED event
arrived in the meantime then nova cancelled the power state update task. No
race, no unnecessary DB update during hard reboot, but if the VM crashed
suddenly, its power state will be updated in the DB *eventually*.

The catch is, Eventlet allows cancelling a task that was started with a delay
specified, native threading does not have such capabilities out of the box.

The naïve implementation with native threading is adding a sleep before the
power state update function in the task. But that is not good. A task running
in a native thread cannot be cancelled after it started running, even if
"running" here actually means executing `time.sleep()`. On the other hand
Executors have a tasks queue to handle the case where there are more tasks than
free workers. Tasks that are queued waiting (another form of "sleeping") for
workers can be cancelled. This gave us a hint how to solve this. Do not
execute `time.sleep()` but instead wait in a queue.

This led to our first, but probably not our last, Executor customization:
[`StaticallyDelayingCancellableTaskExecutorWrapper`](https://github.com/openstack/nova/blob/7e0f18ff2baaf6aec89eb300b6942c488541da7e/nova/utils.py#L1465).
The enterprisy name tells everything: It is a wrapper for Executors.
(As the mantra says: Prefer composition over inheritance.) It supports
cancelling tasks while those tasks are delayed. The statical delay here is just
a simplification. In our use case every task needs to be delayed with the same
statically defined delay. You will see this simplifies the implementation.

So this is how it works:
* It gets a normal Executor from the outside to use to really run tasks
* It has a queue of delayed tasks waiting to be executed after the delay or
  ignored if it is cancelled before the delay runs out.
* It has an extra thread that detects when the delay runs out and executes the
  task if it is not yet cancelled.

There are a couple of complications.

The client expects a `Future` object when it submits a task. Normally this
`Future` object is created by the real `Executor` when a task is submitted. The
whole point in this wrapper is that it does not submit the task right away for
execution, so it needs to create and return its own `Future` object to the
client to satisfy the `Executor` interface, and then make sure that this
`Future` from the wrapper is connected to the real `Future` returned by the
real `Executor` after the task is submitted after the delay. Fortunately the
[`Future` class](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Future)
exposes the `set_result()` and `set_exception()` functions to manipulate the
internal state of a `Future`. So connecting the two `Futures` is as simple as
wrapping the original task into:
```python
    @staticmethod
    def _task_wrapper(task) -> None:
        """This wraps the original task so when it finishes in the real
        executor the result of the task can be copied to the Future object
        already returned to our caller from submit_with_delay(). So
        the caller can get the result or exception from the task.
        """
        try:
            task.future.set_result(task.fn(*task.args, **task.kwargs))
        except BaseException as e:
            task.future.set_exception(e)
```

The `Future` class also exposes the `set_running_or_notify_cancel()` function
that allows our Executor wrapper to atomically check if the client cancelled
the task, otherwise it sets the task to running state and therefore marks it
non-cancellable anymore.

After all this setup the logic of the scheduler thread in the wrapper can be
summarized as follows:
1. Take the oldest task from the queue, if the queue is empty wait on it.
2. Wait until the delay runs out for the task.
3. Check if the task is not cancelled and set it to running. If cancelled go
   to 1.
4. Wrap the task to connect its result to the already returned Future and
   submit it to the real Executor. Then go to 1.

Of course there are a bit more error handling in the code and also we need to
handle clean shutdown if requested.

Note, it is only valid to take the oldest task from the queue and blindly wait
for its deadline because we know that every task has the same amount of delay
(and because we don't believe in time travel), so the oldest task in the queue
needs to be executed next. In the general case when each task can define its
own delay we would need a priority queue and a bit more logic to handle when a
new task arrives which has higher priority (i.e. closer start time) than the
currently highest prio task. As a followup we want to develop
[this generic implementation](https://review.opendev.org/c/openstack/futurist/+/975508)
so others can use it too.

#### I want a single pool but separate concurrency caps on different task types

This is another "exotic" use case for our Executors in nova-compute. But first
we needed to hit a bug to even figure out we have a problem to solve.

The bug was detected in the VMware CI, probably because it runs tempest with
more parallelism. We saw that parallel VM boots hang until they timed out while
setting up the networking for the VM. It turned out that it was a deadlock on
our default Executor that was used both to run the VM build logic in a task and
the network setup for that VM in a separate child task. As we touched before
our Executor sizes are limited in native threading. The default executor was
limited to 5 workers by our default configuration. When 5 VM build arrived at
the same time, each grabbed a worker to build the VM. So the Executor became
full. Then each build task submitted a network setup child task to the same
default Executor, moved on to do some other logic, then finally started
waiting for the network setup child task. As the Executor was full with the
build tasks, the network child tasks were waiting for free workers and never
got them. Therefore the build task waited forever for the networking task.
[Nova logs](https://github.com/openstack/nova/blob/3be17878dcb42095f3ed3fc2819c2bf776c059b5/nova/utils.py#L614-L619)
when an Executor starts queueing up tasks, this helped figuring out
what happened.

This problem taught us that we should care about parent, child task
relationship if they share the same Executor. If the child is not a fire and
forget task, but a task the parent will wait for, then sharing an Executor can
lead to deadlocks. We had the same bug forever in nova, it was just very hard
to produce it as the default Executor size in Eventlet mode was 1000, so it
would have required 1000 parallel build requests to arrive to the same compute
to cause trouble.

So we
[moved the build task to its own long_task executor](https://review.opendev.org/c/openstack/nova/+/977251)
and kept the default Executor for fire and forget tasks and child tasks that
are not spawning grandchildren. Then we saw that build is not the only long
running tasks that used the default Executor. We had snapshot as well. Then
further looking revealed that even though these task types shared an Executor,
the default one, and will share the new long_task Executor, they have
individual concurrency limits implemented by bounded semaphores.

The nova-compute can be configured to limit the number of concurrent VM
lifecycle operations of a given type. E.g., by default nova-compute allowed 10
parallel VM builds, 1 parallel outgoing live migration, and 5 parallel VM
snapshots. Originally it implemented the build and snapshot case with a single
`GreenThread` pool with 1000 workers and two bounded semaphores, one for build
and one for snapshot. As `GreenThreads` are cheap, we did not have to worry
about having a list of tasks, holding up a GreenThread each, while waiting on
the semaphore for their execution slot. This does not work with native
Threading. We cannot waste one thread for each waiting operation where there
could be hundreds of them.

One can ask why don't we keep them in the message bus waiting. The problem
there is that the same message bus topic is used for other, non-capped
operations, so we have to process the messages from the bus to find those
operations.

The simplest solution would be to have a separate Executor for each operation
type with the configured cap used as the size of the Executor. This way
we would rely on the queue in the Executor to hold the tasks waiting for a
slot. We rejected this idea as it creates too many independent Executors to
manage, also it wastes resources as workers, even if they are idle, cannot
be shared across different operation types.

Right now we went with a hybrid approach. The live migration is kept as a
separate Executor as it has a very different limit than the other two. The
build and the snapshot operations are now forced to share the same cap in
native threading mode and therefore implemented by the single shared long_task
Executor.

In the long run another
[Executor wrapper](https://review.opendev.org/c/openstack/nova/+/975924) can be
developed that maintains a per-task-type cap within a shared Executor by having
task queues, and scheduling queued tasks to slots when another task finishes
from the given type.

Also in the future we might want to come up with a way to track parent-child
task relationships and prevent submitting the child task to the parent's
Executor. Right now only code review prevents us from re-introducing such a
deadlock prone situation.

### When unit testing is harder than tempest
I realized this while herding the nova-compute Eventlet removal patches through
CI. I expected more trouble, more instability. Once manual testing
of the native threading mode in devstack showed that the service works, tempest
showed limited amount of surprises. On the other hand we still have unit tests
today that randomly hang or produce strange SQLAlchemy issues when running
with native threading and therefore disabled in the native threaded tox job.
In retrospect I have the following understanding explaining this:
* The tempest suite contains pure black box tests, Eventlet is "just" a
  refactoring, the black box test does not care about our rewiring of internals
  of that box.
* Our unit tests in many cases test the implementation of the unit not the
  behavior of the unit making them fragile during refactoring.
* In some cases unit tests are trying to cover concurrent behavior with
  clever injection of locks and events. Eventlet executes the tasks until
  completion or explicit yielding. Some tests were relying on this
  run-to-completion semantic too which does not exist in native threading.
* Some of our unit tests do too much. E.g., a big chunk of our unit tests
  depend on a DB being provided to the test case.

It will be interesting to see how hard it will be to make our functional tests
pass and remain stable with native threading as this test suite sits in between
tempest and unit test.

### Scale testing

One of the key changes during Eventlet removal is the cost of concurrency. A
native thread costs a lot more memory than a GreenThread. This forced us to
limit our Executor sizes to reasonable numbers as our old defaults were simply
too much. But what is a reasonable default? As you saw above we have many
different concurrent tasks types, limits, and delays. So, it is very hard to
tell analytically what is a reasonable Executor size. Therefore we decided to
try to find it via performance testing.

In the upstream CI we have limited resources (CPU cores and RAM), so we cannot
create big deployments to find out how they scale when the concurrency mode is
changed. We have to make some compromise.

The most resource consuming part of our deployment are compute nodes and VMs
created on them by nova. So if we accept that we cannot scale test nova-compute
then we can limit our resource usage. Fortunately nova already provides a
way to simulate most of the heavy lifting of the nova-compute service by
implementing a fake virt driver. That driver does not spin up VMs on the
hypervisor, it only has an in-memory state, and implements the happy path for
the VM lifecycle operations. With a bit of
[tweaking](https://review.opendev.org/c/openstack/devstack/+/962362) in
devstack we can create 3 cells and 20 compute nodes within a single CI worker
machine. This allows us to scale-test at least our APIs, scheduler, conductor,
and even nova-compute's virt-driver-independent pieces like the
ComputeManager and the ResourceTracker.

Another complication in our CI is that two consecutive job runs will observe
significantly different performance from the CI worker machine. It is because
we use multiple cloud providers, each having different performance and ever
changing load situation. So comparing perf-scale results across multiple job
runs is not feasible as we cannot control the underlying machines. Instead, we
run the same performance tests twice in the same job, once with Eventlet and
once with native threading, then we compare the results within the same job.

This also has the nice side effect that we, at least partially, test the
scenario of a deployment being reconfigured from Eventlet to native threading.

We started using Rally to
[implement performance test scenarios](https://review.opendev.org/c/openstack/nova/+/960130).
We also created an Ansible role that can collect and compare the perf data:
CPU, memory usage, and test execution time. Rally is nice as it gives us
a good framework to implement and execute these tests. But it is not actively
used in nova yet and therefore it lacks nova API microversion support,
and probably other nova specific functionality. So we also started adding
[microversion support to Rally](https://review.opendev.org/c/openstack/rally-openstack/+/960052).

This was ongoing work in Flamingo but due to limited time we stopped it during
Gazpacho to refocus on nova-compute itself. Now that nova-compute is done
(+1 famous last word) we want to go back to this trial and take it further.
The end goal is a job that at least runs periodically and fails if
the native threading performance starts differing significantly from the
Eventlet one.
