---
title: Eventlet removal - Flamingo PTG
date: 2025-04-11 15:30:00 +0200
categories: [OpenStack, Eventlet]
tags: [openstack, nova, eventlet, upstream]
hidden: true
---

This is the first installment of a series of posts talking about the Eventlet
removal effort form the OpenStack Nova project. I will try to use these posts
to summarize any recent progress and plan out the next steps.

## Flamingo PTG

Of the week of the 7th of April 2025 the OpenStack community ran the last
[virtual Project Team Gathering](https://ptg.opendev.org/) to plan out the work
for the next 6 months. One of the topics, and a long-running [community
goal](https://governance.openstack.org/tc/goals/selected/remove-eventlet.html),
discussed there was how to remove the Eventlet library usage from the
codebase.

### Cross project discussion

The discussed topics captured on an
[etherpad](https://etherpad.opendev.org/p/apr2025-ptg-eventlet) and the meeting
recordings([part1](https://youtu.be/uwtDJpATPik),
[part2](https://youtu.be/Epq9EaePzIM)) are available on YouTube.

My main takeaways form the discussion (in revers order):

### Supporting multiple concurrency backends in the same service

The Nova project got requests from operators to temporary support both the
legacy Eventlet, and the new native Thread concurrency model for the services
so that:

* The software upgrade can be done without switching the model and then the
  model can be switched service by service independently.

  We foresee that the new model will rely on a set of fixed sized thread pools
  where the size needs to be tuned based on the given cloud architecture and
  workload pattern. So allowing this tuning to happen independently form the
  upgrade and done service by service helps to remove some risks for the
  deployer.

  The new need of tuning the pool size is coming from the fact that creating
  `GreenThread`s are basically free, so we don't needed to limit it. But
  creating native threads are more expensive, so we cannot have unlimited
  pools.

* If a blocking bug is found in the new concurrency model, which we foresee
  will inevitably happen in such a big change, then there should be a way to
  move back to the legacy concurrency model until the bug is fixed.

During the discussion we also realized that to be able to keep our CI green
while we are migrating to the new model we need to be able to run half
transformed services. This means that the half transformed service needs to
work at least with one of the concurrency models. As today each service works
with the Eventlet model keeping the Eventlet model working all the time is the
way forward. At least until we have the given service fully working with the
threading model. The other alternatives would be either allowing to
break the CI for a while, or keep the changes on a feature branch and do a big
bang integration. Neither of them the Nova team believes in.

During the discussion we also learned that Glance successfully followed this
pattern and supports both models. Also, some Neutron services also already
transformed and supporting both models.

> So we agreed that Nova will also try to keep both Eventlet and the native
thread concurrency model supported at least in that single release where the
full native thread support is first made available.
{: .prompt-info }

### The new `oslo.service` threading backends

The [patch](https://review.opendev.org/c/openstack/oslo.service/+/945720)
for the `oslo.service` library to support native threads as selectable
backend has been proposed and expected to land early in the Flamingo cycle.
There is also an independent
[doc patch](https://review.opendev.org/c/openstack/oslo.service/+/940664)
proposed.

This is a hard dependency for Nova to finish the migration but not a hard
dependency to start the work.

This patch introduces a way to select between the legacy Eventlet backend
and the new native thread backend via a new `init_backend` call:

```python
from oslo_service import backend as service

service.init_backend(service.BackendType.THREADING)
```

This can only be called once and has to be called very early, basically
before any other component (messaging, logging, config) is initialized.

As Nova agreed to keep both backend supported we need a way to select the
backend based on external input. Nova cannot use a configuration option as the
config parsing cannot happen before the backend initialization. We discussed
two alternatives:

* Using different console scripts (i.e. entry points) for the different modes,
  e.g. `nova-compute` and `nova-compute-eventlet` where the entry point hard
  codes the given backend initialization. It has the downside that container
  based installers would require carrying two containers or at least injecting
  two different container configuration to allow switching between the models
  for each service

* Using an environment variable and using that as the input for the backend
  selection.

> So we agreed to use an env variable to select the backed
{: .prompt-info }

### Nova specific discussion

The discussion captured on another
[etherpad](https://etherpad.opendev.org/p/nova-2025.2-ptg#L582).

> Top of the agreement noted in the cross project section we agreed that it
  is OK to do the transformation service by service. So it is OK if some
  services are ready to run in threading mode while others are still only
  supporting Eventlet in the Flamingo release.
{: .prompt-info }

We touched on switching the default to `true` on the
`[quota]count_usage_from_placement` config options to avoid one code path that
calls scatter-gather. However, the two quota modes are not fully equivalent.
So this is controversial. We anyhow need to change our scatter-gather as it
is use in many other places. So I don't think I will try to do this in
Flamingo.

I had a question about moving some code around to make the location of the code
less confusing:

> * move setup_instance_group form scheduler/utils under conductor/utils as
  this is only called by the conductor service.
> * move libvirt/machine_type_utils under cmd/utils as it is only called from
  CLI commands and never from the libvirt driver or the nova-compute service.

There was push back as this move would poison `git blame`. I might still try
to push for it later but definitely not putting it to the critical path.

> We agreed to re-use the existing
  [blueprint](https://blueprints.launchpad.net/nova/+spec/eventlet-removal-part-1)
  and use the global
  [eventlet-removal](https://review.opendev.org/q/topic:%22eventlet-removal%22)
  Gerrit topic.
{: .prompt-info }

> As an extension of the support both concurrency modes agreement we discussed
  and agreed to use two sets of Tempest jobs to test the two concurrency modes.
  However, we are not trying to have two sets of functional test, instead
  we switch the test to run services in native threads in the functional env.
{: .prompt-info }

We also discussed plans and tasks I will not reiterate here but save for a
later section and for later blog posts.

## Where we are today in Nova

## Immediate next steps
