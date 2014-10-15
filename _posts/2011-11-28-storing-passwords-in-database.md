---
layout: post
title:  "Storing passwords in database"
date:   2011-11-28 11:00:00
tags: security
language: EN
---
Security is tricky subject. In fact, we should never think we know about security. The way we store passwords in database is an example of how we might do things the wrong way. We may think storing the MD5 hash of a password, but this is very unsafe as an attacker could use a rainbow table to retrieve the password.

Here are two interesting entries on Stack Overflow:

- [Best way to store password in database](http://stackoverflow.com/questions/1054022/best-way-to-store-password-in-database)
- [How does password salt help against a rainbow table attack](http://stackoverflow.com/questions/420843/how-does-password-salt-help-against-a-rainbow-table-attack)

What they recommend is storing the salted hash of the password. The salt should be different for each password, and it should be a random ASCII string stored along with the password.