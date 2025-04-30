---
title: Eventlet removal - Scheduler first run
date: 2025-04-30 15:00:00 +0200
categories: [OpenStack, Eventlet]
tags: [openstack, nova, eventlet, upstream]
mermaid: true
---
This is a mid-week update on the Eventlet removal work as we hit a milestone.
We had a first green tempest run with nova-scheduler running in native threaded
mode.

You can browse the rest of the blog series
[here](https://gibizer.github.io/categories/eventlet/).

## What happened

Pulling in multiple hacks we had the first
[CI run](https://zuul.opendev.org/t/openstack/buildset/f495c3661ca346a198c80a01387d6ba4) where the nova-scheduler ran in native threading mode and multiple tempest jobs
(e.g.
[nova-next](https://zuul.opendev.org/t/openstack/build/736ff6efec4c4a22a103401bcd321d8d), [nova-multi-cell](https://zuul.opendev.org/t/openstack/build/636bdbb6ad494ef0b49415034842a640))
succeeded.

It needed 3 sets of patches:

* The nova series
  ([tip](https://review.opendev.org/c/openstack/nova/+/948311/8)) that:

  * Translated the scatter-gather logic to the futurist
    based abstractions.
  * Introduced `ThreadPoolExecutors` as the only way to spawn concurrent tasks
  * Added logic to switch between Eventlet based `GreenThreadPoolExecutor` and
    the native thread based `ThreadPoolExecutor` based on an environment
    variable passed to the service.

* The `oslo.service` series that consists of two patches:

  * the work-in-progress official
    [threading backend implementation](https://review.opendev.org/c/openstack/oslo.service/+/945720/11) for `oslo.service`,
  * and a
    [patch](https://review.opendev.org/c/openstack/oslo.service/+/948310/8)
    with a set of hacks to make the feature good enough for the nova-scheduler
    to start up.

* The
  [devstack patch](https://review.opendev.org/c/openstack/devstack/+/948408)
  that hard-coded the environment variable passing to nova-scheduler to switch
  it to threading mode.

With this setup nova-scheduler successfully spawn two independent worker
processes each using native threads to:

* handle incoming RPC messages to schedule VMs
* run scatter-gather to get all the `HostState` objects from multiple Cell
  databases

We added a semi periodic log to report the `ThreadPoolExecutor`'s state. This
example is from the
[n-sch service logs from the nova-multi-cell run](https://zuul.opendev.org/t/openstack/build/636bdbb6ad494ef0b49415034842a640/log/controller/logs/screen-n-sch.txt#18588-18589):

```log
Apr 29 14:36:59.159065 np0040575581 nova-scheduler[84912]: INFO nova.utils [None req-7b241ea0-60c2-4dc7-af64-ef5760dd07ea tempest-TestSnapshotPattern-434630328 tempest-TestSnapshotPattern-434630328-project-member] Stats of scatter-gather executor: <ExecutorStatistics object at 0x7466d85eb7c0 (failures=0, executed=218, runtime=4.67, cancelled=0)>
Apr 29 14:36:59.159282 np0040575581 nova-scheduler[84912]: INFO nova.utils [None req-7b241ea0-60c2-4dc7-af64-ef5760dd07ea tempest-TestSnapshotPattern-434630328 tempest-TestSnapshotPattern-434630328-project-member] State of scatter-gather ThreadPoolExecutor: max_workers: 40, workers: 40, idle workers: 40
```

This shows us that the given worker process (we had two) executed 218
scatter-gather tasks in the thread pool with 40 worker threads. More
importantly each worker thread got back to the pool and become idle. So we had
no hanging or lost threads.

The
[memory usage of the worker processes](https://zuul.opendev.org/t/openstack/build/636bdbb6ad494ef0b49415034842a640/log/controller/logs/screen-memory_tracker.txt#4831-4832)
seems acceptable, both process reported around 122 MB RSS allocation at the end
of the job run:

```log
Apr 29 13:16:59.157626 np0040575581 memory_tracker.sh[104836]:      84924    1.5          122160      84038   00:00:03       69 do_select                 nova-scheduler: ServiceWrapper worker(1)
Apr 29 13:16:59.157626 np0040575581 memory_tracker.sh[104836]:      84912    1.4          121776      84038   00:00:02       67 do_select                 nova-scheduler: ServiceWrapper worker(0)
```

This is higher than in Eventlet mode, 56 MB RSS (in a different run of the same
nova-multi-cell job), but we expected this and planning to fine tune the
number for worker threads from the default 40 (cpu * 5) to a lower default
value and allow changing the default via configuration. We assume that most of
the deployments use less than 5 cells, so a pool size of 4 feels reasonable.
We will see.

The execution time of the job seems 10% higher, but we have high variability
in our CI, the scheduling time is a small component of most of the tempest
tests, and we have a single measurement from the threaded scheduler. So I
consider this value acceptable for now, and we will collect more data as we go.

There is also
[work in progress in devstack](https://review.opendev.org/c/openstack/devstack/+/948436)
to allow passing environment variables to systemd service files. This allows us
to
[turn on the threading mode for a given service in a specific job from the zuul config](https://review.opendev.org/c/openstack/nova/+/948450/2/.zuul.yaml#493).

## How to reproduce it

In CI you can use the
[patch](https://review.opendev.org/c/openstack/nova/+/948450/2) as the base of
your patch and then CI will run the nova-next job with nova-scheduler in
threading mode.

Or you can start you local devstack with a `local.conf` where `oslo-service` is
in `LIBS_FROM_GIT` and then pull the
[oslo.service patch](https://review.opendev.org/c/openstack/oslo.service/+/948310/8)
and the [nova patch](https://review.opendev.org/c/openstack/nova/+/948311/9)
into your devstack. Then stop the nova-scheduler process and start your own
from a terminal with
`OS_NOVA_DISABLE_EVENTLET_PATCHING=true nova-scheduler --config-file /etc/nova/nova.conf`.

If you see logs like `State of scatter-gather ThreadPoolExecutor:` then you
can be sure that the nova-scheduler runs in threading mode.

## Why this is important

This shows that our generic idea to translate things to the futurist lib and
then use Eventlet or native threads implementation behind that lib selectively
works well enough to continue translating more code and doing more non-happy
path testing with the current PoC.

This also shows that the memory consumption and runtime of the threaded code
is not terrible.

Overall this removed a good chunk of uncertainty and risk from our technical
plans and provides useful patterns for other projects to copy. In the process
we were able to provide actionable feedback from a realistic env for the
`oslo.service` threading backend implementation as well.
