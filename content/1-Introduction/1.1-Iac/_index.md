---
title : "Infrastructure as Code"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1.1 </b> "
---

Typically, most beginners with Cloud in general, and AWS in particular, interact with AWS services through the console. They log in to the console and work with the services they need. This process is quite straightforward and convenient if we are only creating a few small applications without the need to reuse those configurations.

Simply put, we can understand by its name that we will write code to describe and manage our infrastructure. **Infrastructure as Code (IaC)** allows us to interact with those services through lines of code, rather than manually interacting with the console.

**Advantages:**
- Can be reused to create other applications with similar infrastructure, reducing the time needed to create infra.
- Provides a clear structure and tracks the state of the created infrastructure. Imagine a large system where you’ve created the infra, but the next day you can’t remember whether you’ve done it or not.
- Can back up the infrastructure in case of system failures.
