---
layout: post
title: "That DevOps Thing"
description: "Wherein I try and fail to explain why SysAdmins have to program"
category: DevOps
tags: [DevOps, SysAdmin, Developers, Programming, Teams, Product]
---
{% include JB/setup %}

When I started my first gig outside of hosting, it was walking into a different
world. Until then, my exposure to programming endeavors usually consisted of quick
and dirty one liners, kick scripts, off the shelf ticketing systems for support and
billing, and if you were lucky, a backend system poorly written in the interpretive
language du jour for asset/inventory tracking.  The dev team, if there was one, did
no modern code practices like QA, review, or sprint planning, and most
frustratingly answered to no one. Any customer facing issues that were due to bugs
were usually ignored and fell to the wayside. Not to say that things never
improved. Eventually, product people were brought in to turn business requirements
into actual coding projects.  Project Managers organized sprints and kept the Devs
on task. QA people tested code and enforced guidelines. When I started at my
current place of employment though, this was the norm and not a brand new practice.

So, the one thing I needed to figure out: How did SysAdmins like me and other Ops
people (NetAdmins, DB Admins, etc.) fit into the picture? We usually work in
proximity to the devs, but just keep the lights on so they can code away.

DevOps is typically regarded as more of an idea than something concrete. Most
people like to explain it as an intermingling of Dev and Ops, two jerks supping
on peanut butter and chocolate colliding into each other yet ultimately finding
the whole experience enjoyable. Think of it like the Final Fantasy job system.
Sure, your job title may say "Backend Developer", "Sysadmin Administrator", or
"QA Manager", but in DevOps land, you are able to freely level up in other
'classes'. From a managerial standpoint, it must be awesome to get your team to
take on more responsibilities without having to pay anyone extra.

Generally, what it boils down to is that Devs are suddenly spinning up their own
local/test environments and aren't completely dependent on DevOps for every little
thing that's systems related. Meanwhile, Ops people are treating their
infrastructure as a coding project. The best part of this is that the team members
eyes on the other side of the aisle no longer gloss over when you talk about
inheritance or packet inspection.

As you can tell from the blog title, I will likely be covering more of the Ops side
of DevOps things, and how a regular Ops person can pick up Dev practices to work
more efficiently.  Next time, I will touch on treating your infrastructure as a
code project, starting with the planning stages.
