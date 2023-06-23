---
title: Telemetry with Rust Part 1 - The basics
date: 2023-01-08 13:00:00
categories: [Rust]
tags: [rust]
---

This blog post explores the basics of the TLS (Transport Later Security) protocol with Rust.
Have you ever wondered how those certs work exactly and how to use them? In this post we will create our own certificates, our own CertificateAuthority(CA) and sign them. Then we will use them in a very simple client-server application written in Rust.

> Disclaimer: This post will not explore the internals of any cryptographic algorithm
{: .prompt-danger }
