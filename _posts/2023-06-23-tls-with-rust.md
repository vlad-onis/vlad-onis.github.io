---
title: TLS basics
date: 2023-01-08 13:00:00
categories: [Rust]
tags: [rust]
---

This blog post explores the basics of the TLS (Transport Later Security) protocol. Have you ever wondered how those certs work exactly and how to use them? This post explores the theoretical concepts behind TLS, made easy, and it provides you with a TLS walkthrough Rust application.

> Disclaimer: This post will not explore the internals of any cryptographic algorithm
{: .prompt-danger }

## Introduction

The TLS protocol is extensively used in web communication. Most websites nowadays implement the HTTPS protocol which encapsulates the TLS. It's hard to pin TLS's place in the OSI stack but in the TCP/IP stack I would place it somewhere between the Transport Layer and the Application Layer.
In order to fully understand it we will need to check out the following subjects:
* Symmetric encryption
* Assymetric encryption
* Public key cryptography
* TLS handshake 

The last section of this blogpost (Hands on TLS) contains follow along bit with Rust. Feel free to skip ahead if you are familiar with the high level concepts of TLS.

## Symmetric encryption

The concept is really simple and detailed in the diagram below. Both parties involved in the communication use the same key (hence the name symmetric) to encrypt and decrypt the payload. There are various algorithms that are used, for example AES. 

<!-- ![Symmetric_Communication](/assets/img/tls_basics_resources/symmetric_communication.png) -->
<img src="/assets/img/tls_basics_resources/symmetric_communication.png" height="150%"/>

**Open questions:**
<span style="color: red;>"1. How will we exchange the symmetric key so both parties have it but no man in the middle has it?
2. Assuming both parties have the key and no one else, how will we protect the message from reflection attacks, or from attackers that modify the cipher text without decrypting it.</span>

This model alone is not enough to ensure secure communication but it has the benefit that once the keys are exchanged, it is very performant compared to other solution

## Asymmetric encryption

This type of encryption is based on 2 keys for each party instead of just one which solves the key exchange problem. But this kind of algorithms are normally way slower than symmetric encryption so they are not used for bulk data encryption but generally just in the key exchange process.
Won't be detailed here but such algorithms that perform secure key exchanges are:
* Diffie Helman
* RSA
* ECC
* DSA

Usually Asymmetric and Symmetric encryption algorithms are used together in the following way:
-> Asymmetric encryption takes care of the key exchange so both parties end up with a "session" key. 
-> The "session" key is then used to encrypt/decrypt bulk data using symmetric key algorithms such as AES(for example).

Note that key length and most parameters really are sensitive and prone to attaks, so don't implement these in production yourself - for learning only.

## TLS
Finally, right? What could there be more to do to gain the trust of the other party in a communication. Turns up that trust is a messed up concept, hence the protocols are messed up as well (but smart nonetheless).

TLS uses the concept of **digital certificates** to ensure a secure and integrous communication channel between 2 parties. What the heck is a certificate? To understand that we need to first look at Certificate Authorities

### Certificate Authority(CA) and Digital Certificates
They are trusted entities that issue and manage digital certificates. From all the responsibilities of a CA we are most interesting in issuance and signing of the certificates. Besides those a CA can also revoke certificates or verify the identity of a certified identity.

Certificate signing: Usually the server makes a CSR (Certificate Signing Request) to the CA. The CA then generates some certificates that will contain the server public key and the CA signature. The certificate is signed using CA's private key to ensure integrity of the certificate (public key cryptography technique).

After the server obtains a signed certificate it will send that to any client that wants to communicate securely to the server. The client will be able to get the public key of the server out of the certificate and then perform the key exchange described in the previous section. At that point the secure channel is established. All these steps are called the TLS handshake, which is both robust and performant because it is only performed once.

How comes the client trusts the CA. Well when the CA signs the certificate for the server (the leaf certificate) it also attaches its chain of certificates (signed by other authorities). The root node in the chain is signed by the root CA which is self signed and represents the ultimate trust anchor that all the operating systems and web browsers trust. The client is able to verify the whole certificate chain because the root CA is trusted by his operating system, going down the line and validating each certificate in the chain will yield the leaf certificate and the server public key.

The diagram below oversimplifies the protocol to better understand the concept.

![TLS_Handshake](/assets/img/tls_basics_resources/tls_handshake.png){:height="1000" width="1000px"}

## Hands on TLS

As this post is becoming quite long I will refrain from pasting the entire code here. For those of you that want to explore generation and usage of digital certificates I prepared a little walkthrough **[here](https://github.com/vlad-onis/tls_showcase)**. Follow the reame step by step and by the end you will be able to generate, sign and use certificates in 2 methods (mkcert and openssl).




