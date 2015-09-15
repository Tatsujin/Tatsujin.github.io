---
layout: post
title: "No More Tiers"
description: "When planning an infrastructure project, think on tiers"
category: DevOps
tags: [DevOps, Infrastructure, Tiers, Architecture, Planning]
---
{% include JB/setup %}

When planning a programming project, it helps to have a high-level, abstract plan
to simplify things. The nice thing about planning server architecture is that it
is somewhat standardized.  You can visualize the project as a series of tiers.
We'll go over them here

## Ground Rules
* If someone else does it better, use it - It is usually more cost effective
to have third-parties handle services, such as outgoing mail, incoming mail,
monitoring, metrics, etc.
* If you can generalize, do it - Infrastructure as a Service (IaaS) providers can
be very effective, but make your infrastructure plan in a way that you can easily
move from one provider to another for a given service. The challenge here usually
stems from high level services (DNS, CDN, Private Networking, etc.). Stick to
internal standards, and have a major plan if possible for major IaaS providers.

## Public DNS Tier
The public DNS tier is usually reserved for production and production-like
environments. Generally, this tier is used in concert with the CDN tier, as many
CDN providers have discovered they can use their resources in speed/redundancy
for DNS as well.

## CDN Tier
Again, the CDN tier is usually reserved for production and production-like
environments. Generally, you will want to optimize for loading time and bandwidth
savings at this level.

## Origin Tier
The origin tier is usually a load balancer, the layer that terminates HTTPS traffic.
It may not though, if you require full security within the private network. It is
useful to have your non-production environments start here to ensure that load
balancing is working correctly.

## Web Tier
In some cases, the Origin Tier can be merged with this layer, because most major
load balancers are reverse HTTP/S proxies and can be used as such. However, some
organizations might find it useful to operate their own Web Tier to ensure that
actions such as redirects and custom HTTP -> APP connectors (mod_jk for example)
are used.

## App Tier
This usually comprises of an application server that is running the code for the
application(s) that your devs will be developing. Generally, a local test environment
will start here so developers can rapidly verify their changes.

## Cache Tier
This tier usually consists of a caching application (memcache, redis, etc.) that
can keep session data even if Application servers are removed/added, ensuring that
a user's ephemeral data stays on the application as long as possible. It makes sense
to keep short term backups of cache data in case of failure.

## Data Tier
This is often regarded as the most important tier and most impossible to keep
stateless. This is usually a full relational database, although newer applications
may use more simplified models (MongoDB, etc).

## Service Tier
This is generally considered as more of an ephemeral tier, as it covers a litany
of side services, such as:

* Outgoing Mail
* Monitoring
* Log processing
You will usually want to use third-party services where possible and cost-effective

## Message Tier
This is another ephemeral tier. In larger groups of applications, a messaging system
such as ActiveMQ, RabbitMQ, etc. is used as a buffer between services.  This is
generally utilized to improve redundancy by allowing messages from a higher tier
to a lower one to be queued and kept in place if the lower tier is unavailable.
