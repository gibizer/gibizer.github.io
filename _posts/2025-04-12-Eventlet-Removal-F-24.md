---
title: Eventlet removal - F-24
date: 2025-04-14 18:50:00 +0200
categories: [OpenStack, Eventlet]
tags: [openstack, nova, eventlet, upstream]
---

This is the next installment of a series of posts discussing the Eventlet
removal effort from the OpenStack Nova project. We have 24 weeks left from the
Flamingo cycle.

You can browse the rest of the series
[here](https://gibizer.github.io/categories/eventlet/).

## Current state

We are just after the PTG (see my summary
[here]({% post_url 2025-04-12-Eventlet-Removal-Flamingo-PTG %})) and we
have a list of agreements with the team that should guide our work.

There are a couple of up-to-date patches that can be landed right away:

1. [Remove nova debugger functionality](
https://review.opendev.org/c/openstack/nova/+/922496)

   This rips out the remove debug feature of the nova services. As far as we
   know this was unused and cannot really be used to meaningfully
   debug Nova services today. The warning emitted by the
   [code](https://review.opendev.org/c/openstack/nova/+/922496/7/nova/debugger.py#b57)
   explains the problem with the debugger pretty well:

   ```python
   LOG.warning('WARNING: Using the remote debug option changes how '
               'Nova uses the eventlet library to support async IO. This '
               'could result in failures that do not occur under normal '
               'operation. Use at your own risk.')
   ```

   Maybe after we finished switching Nova to the native threading model we can
   reintroduce a remote debugger in a better way.

2. [Split monkey_patching form import](
https://review.opendev.org/c/openstack/nova/+/922425)

    Instead of monkey patching during importing a module monkey patching now
    called explicitly after the import. This allows us to better see when
    we are monkey patching. This patch shows that we have 4 ways to enter
    nova code and got monkey patched:

    1. `nova.cmd` - used by all of our CLI commands and non WSGI services
    2. `nova.api.openstack` - used by our WSGI services
    3. `nova.test` - used by our unit test environment to run nova services in
       GreenThreads.
    4. `nova.tests.functional` - used by our functional test environment to run
    nova services in GreenThreads.

## Next steps

We identified some immediate steps that we can do right away while in parallel
we continue breaking down the whole problem into smaller steps.

### Deprecate usage of `oslo_service.wsgi` / Eventlet base WSGI

On the PTG we heard from the `oslo.service` folks that there won't be any
replacement for `oslo_service.wsgi` in the threading backend. This
sparked a discussion in Nova if this effects us or not and agreed to deprecate
any not yet deprecated usage of the Eventlet based WSGI server.

Nova [deprecated](
https://github.com/openstack/nova/commit/b53d81b03cad73fac7f558d287db0354f0a46ec1)
the Eventlet based WSGI server in Rocky. So we can remove this from Nova in
Flamingo.

Note that Nova does not directly depend on `oslo_service.wsgi` but basically
re-implements the same functionality in
[`nova.service.WSGIServer`](https://github.com/openstack/nova/blob/1ad11b13884baeaa6ed9f8f5818f4d176f4d3134/nova/service.py#L330). So we are not
directly impacted by the `oslo_service.wsgi` removal, but in any case we have
to get rid of our own Eventlet WSGI server.

### Replace Eventlet primitives with equivalent stdlib primitives

There is a list of Eventlet concurrency primitives that behaves the same as
the stdlib counterparts when monkey patched. So we assume that we can simply
replace the Eventlet based primitive with the stdlib one and therefore remove
a lot of direct Eventlet imports.

The replacement we will try shortly:

* `eventlet.sleep` => `time.sleep` 16 times
* `eventlet.event.Event` => `threading.Event` 34 times
* `eventlet.semaphore.Semaphore` => `threading.Semaphore` 2 times
* `eventlet.semaphore.BoundedSemaphore` => `threading.BoundedSemaphore` 1 times
* `eventlet.queue.LightQueue` => `queue.SimpleQueue` 2 times

Probably worth to add some
[hacking checks](https://github.com/openstack/hacking) to prevent adding the
Eventlet based primitives in new patches while we are actively trying to remove
them.

There is a bunch of other Eventlet imports that are not that easy to replace
like `eventlet.spawn`, `GreenPool`, `tpool.Proxy`. So we will take them one
by one later.

### Refine the task breakdown

These are high level directions we need to investigate towards and create
smaller tasks out of it:

* Figure out what it means to rip out the Eventlet based WSGIServer.

* Figure out how to run nova services in threads in the functional environment.

* How to do proper timeout handling in scatter-gather to make it to Eventlet
  independent.

* Group the usage of `eventlet.timeout` and draft a solution for each group.

* Re-architecture the Libvirt event handling thread and our current
  `tpool.Proxy` usage.

* Check the NoVNCProxy (and other console proxies) Eventlet usage and draw a
  plan how to run them with native threading and how to ensure no performance
  regression happens during the change.

... and probably many others currently unknown.
