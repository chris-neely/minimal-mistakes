---
title: "Okta Admin Scripts Part 2 - Import Group Members"
tags: 
  - PowerShell
  - Okta
  - API
published: false
---

## Summary
I highly recommend you read [my previous post](okta-scripts-part-1) if this is your first time working with the Okta API.  Last time I went into a lot of detail about how to generate Okta API keys, find group ids, and other pre-requistes we will not cover in this post.  In this post we are going to cover taking a CSV file of email addresses and adding those corresponding users to an already existing group with the Okta api.

## Lets review the code...
