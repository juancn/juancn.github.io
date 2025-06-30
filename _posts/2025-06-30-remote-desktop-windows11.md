---
layout: post
title:  "Microsoft accounts and remote desktop on Windows 11"
date:   2025-06-30 15:51:00 -0300
categories: post
---

This is a weird one, but I wanted to leave this out there in case it helps anyone else.

First a bit of background, I recently built a new gaming PC and installed Windows 11 Profesional on it, logged in with my Microsoft account
without problems (via pin and email). 

The issue was when I enabled Remote Desktop, the computer would not authenticate remotely with my Microsoft account.

Long story short: you first need to login with your Microsoft password *locally* on the computer.

And how one does that?

You need to go to go Settings > Accounts > Additional Settings and disable "For improved security, only allow Windows Hello sign-in for Microsoft accounts on this device (Recommended)".

Then, you log-out, log in but select "sign-in options" and enter a password.

Once you done that, you can go back and disable password logins again if you like.

From that point on, the account can be used to remote login.
