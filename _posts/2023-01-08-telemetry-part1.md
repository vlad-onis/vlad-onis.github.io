---
title: Telemetry with Rust Part 1 - The basics
date: 2023-01-08 13:00:00
categories: [Rust]
tags: [rust]
---

# This document is still a DRAFT - WIP

This blog post introduces the telemetry concept in Rust as described by Luca Palmieri in his book "Zero2Production". 

> Disclaimer: This blogpost represents my personal notes and explanations of Zero2Production Telemetry chapter and should serve as a summary. No copyright intended.
{: .prompt-danger }


## Introduction

Taking an application to production is not easy, it involves writing the application, testing, packaging and shipping to say the least. But even with all the steps done right your application will be exposed to known and unknown unknowns. That is a mouthfull but let me explain.

Known unknowns are issues that we can think of and fix, such as:
* What happens when we lose connection to our database?
* What happens if an attacker injects a malictious payload in our APIs?
  
We call unknown unknowns, issues that we haven't seen before and we we're not expecting. They usually appear at the crossroad between our software components and "the outside world". 

  * The system is pushed outside it's working conditions (e.g. huge traffic).
  * Multiple components fail at the same time
  * No changes for a long time (e.g. no restarts either).
  * etc


In order to be able to fix unknown unknowns we need observability over our app. Data about our running app that is being collected automatically which will be able to inspect, gives us the upper hand over unknown unknowns.
To be observable we need to:
  * to instrument our app to collect high quality telemetry data
  * To allow tools to slice and manipulate the data to find answers.

## Logging

Logs are the most common telemetry data out there. A log entry represents a bunch of text in a given format (e.g. Json)
The default crate for logging is called log (https://crates.io/crates/log) and it provides 5 macros with different log levels (ascending order)
    * trace(most verbose)
    * debug
    * info
    * warn
    * error

The log crate leverages the facade pattern which answers the question: What should we do with these logs?
The facade pattern is just a fancy way of saying, the crate provides an interface (in this case the Log trait) which hides the complex implementation details of how the produced log entries will be used.

Let's look at a very simple example. Throughout the Telemetry series we will use the same app, so we might write the barebones now. It will be a simple app that tackles the subject of building houses or some other buildings.

Start by creating a new rust binary like so:

```bash
cargo new tracing_buildings
```





