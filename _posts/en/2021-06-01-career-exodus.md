---
lang: en
page_id: career_exodus
permalink: /posts/career-exodus
title: "Exodus: From the Support Queue to Security Engineering"
date: 2021-06-01
categories: [Career, Security]
tags: [Career, Exodus, Security, JavaScript, Ruby on Rails, AWS, Chrome Extension]
author: Angel
description: Two and a half years at Exodus, from answering support tickets to engineering the security tooling that protected wallet users, built by solving the problems I was closest to.
image:
  path: /assets/img/career-exodus.png
  alt: Exodus logo
---

## Exodus: From the Support Queue to Security Engineering

**June 2021 to February 2024 · Exodus Movement Inc. · Fully remote**

I joined [Exodus](https://www.exodus.com/), the self-custody crypto wallet, as a Technical Support Engineer and left as a Security Engineer. The jump didn't come from a job application. It came from building tools for the problems I saw every day in the support queue, until building tools became the job.

| Role | Dates | Focus |
|---|---|---|
| Technical Support Engineer | 06/2021 to 01/2023 | Customer support, internal tooling, QA automation |
| Security Engineer | 01/2023 to 02/2024 | Scam and threat response tooling, SecOps automation |

---

## Starting in support

Support at a wallet company is closer to forensics than it sounds. A user writes in about a missing transaction or a suspicious address, and before you can help, you have to work out which of dozens of blockchains it lives on, find it in the right block explorer, and check whether the address is already known to be malicious. Across a team of more than 100 people, those minutes added up on every ticket.

So I built something to remove them.

### The hackathon project

Exodus ran its first customer support hackathon, the first one in that department to involve code. I led a team and we won with a **ReactJS Chrome extension** that takes any address or transaction hash, matches it to the right chain (61 chains for addresses, 31 for transactions) and checks it against a database of known malicious addresses.

It didn't stay a demo. I kept coding and maintaining it, and the whole 100+ person department adopted it. It brought average response time **under 10 minutes**, a key performance indicator for the team.

The write-up and code: [Cryptocurrency Blockchain Explorer Extension](/posts/chrome-extension).

### More automation

The same instinct carried over to other repetitive work:

- **QA automation.** I automated regression testing of a browser extension with Puppeteer, cutting manual regression scripts by **20%**.
- **Community.** I designed, built and containerized a community Discord bot on Kubernetes, which lifted user engagement by **20%**.

Along the way I studied for it formally: the Blockchain Developer program at Consensys Academy in 2022, and Le Wagon's web development bootcamp in 2023.

---

## Moving into security

By 2023 most of my time went to fighting scams that target wallet users: phishing sites, fake apps, drainer addresses. The tracking lived in spreadsheets, which don't scale when several people work the same case and evidence has to be attached and followed up.

As a Security Engineer, I led the engineering, containerization and deployment of a **Ruby on Rails, JavaScript and PostgreSQL** app for reporting and tracking indicators of compromise (IoCs), running on **EKS in AWS**. It let several people work the same case, connected to several external APIs for reporting, and accepted submissions from people outside the team.

- Threats reaching Exodus users dropped by **50%** within six months.
- Resolution time sped up by **20%**, thanks to the API integrations.

Later I migrated the Ruby app to Node with a React and TypeScript front end, and kept building Chrome extensions and NodeJS libraries to automate QA and SecOps work.

How the app works and how to deploy one yourself: [Deploying SecOps Defense with IoC Database Capability](/posts/secops-data).

---

## What I took with me

- **The best internal tools come from the people who use them.** I knew what to build because I had answered those tickets myself.
- **Measure what the team already cares about.** Response time and threats reaching users were numbers everyone watched, so the tools were judged on them.
- **Security is a product, not a report.** A threat database only helps if people can use it under pressure.

That mix of support empathy, automation and security is what I took into my next roles, including [my current work at ARTSVP](/posts/career-artsvp).
