---
layout: post
title: Distributed Encrypted Deduplicated Backups
---
Recently a customer of mine asked for a backup system with with a few interesting twists. The first was they wanted the backups to be sent to a distributed storage cluster. It also needed to encrypt every backup, deduplicate the chunks and store the encryption keys in a secure area. Lastly it needed to be automated in both backup and restore functionality. 

My first thought was I have conflicting requirements here. Encrypted files even if broken up into small chunks generally don't deduplicate well if at all. If I were able to meet these requirements though it opens up interesting new possibilities. You could now roll your own backup service in the cloud and backup to untrusted devices. It turns out there's a form of encryption called [convergent encryption](https://en.m.wikipedia.org/wiki/Convergent_encryption) that produces idempotent output.  This comes with a downside. It's open to confirmation attacks. I think depending on your use case that may not be a problem. 

After starting on some pseudocode I found a project on GitHub that met almost every requirement except the key storage and the back end cluster support. I forked [Preserve]()