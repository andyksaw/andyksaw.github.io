---
title: "Migrating to Fastlane (Part 1)"
date: 2022-01-06T23:00:26+09:00
tags: [ios, fastlane]
categories: [ios, fastlane]
---

App releases, app distribution and general day-to-day development is full of laborious, manual tasks that squander our time. For iOS developers, Fastlane is an invaluable tool that can help automate away most of this. 

It's fantastically easy to integrate into your workflow - that is, unless you find yourself with an app that has already seen multiple, manually-performed releases. I found myself in said situation recently, banging my head in frustration at the lack of documentation regarding this. Fastlane is simple if you add it in from the beginning (and most guides assume you'll be doing so), but full of gotchas and landmines later if you don't. 

This series of posts outlines the process I went through and the various caveats I discovered along the way. By the end of it, we'll have an automated app-release pipeline that we can be proud of.
 ap

* Part 1 - App Store Connect API Key for authentication
* Part 2 - Storing certificates/profiles in a GCP Bucket
* Part 3 - Reusing existing certificates, profiles and keys
* Part 4 - Migrating to manual app signing
* Part 5 - Provisioning Profile device list
* Part 6 - Uploading to TestFlight

# Part 1 - App Store Connect Key

We'll start off with the lowest hanging fruit: switching our authentication to using an API token.

Any operation performed by Fastlane that involves the App Store (eg. submitting to TestFlight, releasing the app) or certificates/profiles (eg. downloading, updating, etc) hits an App Store Connect API under the hood. For obvious reasons, these operations all require authentication to prove that you are allowed to do so.

## Credentials: The Old Way

If you've been releasing an app manually, you would ordinarily log-in to App Store Connect with your email address and password in the browser.

Fastlane uses this as the default form of authentication, and it is __not recommended due to enforced Multi Factor Authentication__. In other words, the app owner's account must have (Apple's) 2FA enabled, and therefore it will be linked to devices of that account. If you are not said person and need to release the app (manually or via Fastlane), you will need to pester said person often for a generated 2FA code. 

For the same reason, CI/CD will not be possible if you use this way, as it's infeasible to input an always-changing code on every job.

Frustrating? Yes. Let's fix that first for our sanity.

## Private Key: The Better Way

A key on the other hand will not prompt you for a 2FA code. As with any API token, you must take care who has it, as it grants the holder the ability to hit the APIs.

A token can be generated on App Store Connect.


WIP


> ⚠️ Your account must have the __Admin__ role to generate a token (or even see a link to the page). The __App Owner__ role is not sufficient.

> 🤯 App Store Connect occasionally has weird caching issues. If your role was just updated to Admin but you still don't see the tab to create the token, try logout of App Store Connect and restart the browser.