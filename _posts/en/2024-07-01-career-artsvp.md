---
lang: en
page_id: career_artsvp
permalink: /posts/career-artsvp
title: "ARTSVP: Building the Event Platform Behind Art World Openings"
date: 2024-07-01
categories: [Career, Rails]
tags: [Career, ARTSVP, Ruby on Rails, JavaScript, PostgreSQL, DevOps, Security]
author: Angel
description: Full stack engineering at ARTSVP, the event and booking platform for galleries, art fairs and cultural institutions, from product features to infrastructure, performance and security.
image:
  path: /assets/img/career-artsvp.png
  alt: ARTSVP logo
---

## ARTSVP: Building the Event Platform Behind Art World Openings

**July 2024 to present · ARTSVP · Full Stack Engineer · UK/EU remote**

[ARTSVP](https://artsvp.com/) runs the guest side of the art world: invitations, RSVPs, bookings, ticketing and check-in for galleries, art fairs and cultural institutions. When a gallery opens a show or an art fair opens its VIP preview, ARTSVP decides who gets in, when, and how smoothly.

I joined in July 2024 as a Full Stack Engineer. Before that I spent a few months at Emma Sleep maintaining Go microservices and Java SaaS components for their e-commerce platform, after [two and a half years at Exodus](/posts/career-exodus).

| | |
|---|---|
| **Role** | Full Stack Engineer |
| **Since** | 07/2024 |
| **Stack** | Ruby on Rails, Hotwire and Stimulus, JavaScript, PostgreSQL, Sidekiq, Stripe |
| **Areas** | Product features, database performance, DevOps, monitoring, code review, security |

---

## What the platform does

The product looks simple from the outside and isn't underneath:

- **Events and bookings**, including timed slots, capacity and waitlists.
- **Invitations and guest lists** that institutions curate carefully and that have to stay private.
- **Exhibitor portals** that let galleries at an art fair manage their own guests under the fair's event.
- **Payments** for ticketed events, through Stripe.
- **QR check-in** at the door, which has to work on opening night with a queue outside.

It's multi-tenant, so every organisation sees only its own data, and organisations connect to one another (a fair and its exhibitors, an institution and its partners).

---

## My work

### Full stack product work

I build features end to end in Rails and JavaScript, from the data model to the UI. My guiding idea is that the best software feels thoughtful: small, contextual features that notice what someone is trying to do, use context the platform already has, and offer the next step instead of leaving them to find it.

### Performance and infrastructure

I lead much of our infrastructure and optimisation work: database tuning, query and index work on the busiest pages, background jobs, deploys and monitoring. For an events platform, the hard moments are predictable (invitations going out, doors opening), so performance work targets those spikes.

### Security

Security has followed me from [my time at Exodus](/posts/career-exodus). At ARTSVP I run SecOps review and auditing, and that work has stopped several live threats, **including a supply chain attack**. I also keep a close eye on privacy boundaries: guest lists are sensitive, and an exhibitor should never see another gallery's guests.

### Working with AI tools

A lot of my day runs on a local, memory-backed AI coding setup: parallel agents in isolated workspaces, a knowledge graph of the codebase, and notes that carry decisions across sessions. I wrote up the whole stack in [The Ultimate Local AI Coding & Memory Stack](/posts/developer-setup-ai-tools).

---

## Why I like this work

ARTSVP combines things I care about: art, careful product design and the engineering underneath. The people using it are running live events, so reliability and privacy aren't abstract. When check-in works on a busy opening night, nobody notices the software, and that's the goal.
