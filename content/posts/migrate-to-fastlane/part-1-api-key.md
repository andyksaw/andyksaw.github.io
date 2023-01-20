---
title: "Migrating to Fastlane (Part 1)"
date: 2022-01-06T23:00:26+09:00
tags: [ios, fastlane, app store connect, api key]
categories: [ios, fastlane, guide]
---

App releases, app distribution and general day-to-day development is full of laborious, manual tasks that squander our time. For iOS developers, Fastlane is an invaluable tool that can help automate away most of this. 

It's super easy to integrate into your workflow - that is, unless you find yourself with an app that has already seen multiple, manually-performed releases. I found myself in said situation recently, banging my head in frustration at the lack of documentation regarding this. Fastlane is simple if you add it in from the beginning (and most guides assume you'll be doing so), but full of landmines later if you don't. 

This series of posts outlines the process I went through and the various caveats I discovered along the way. By the end of it, we'll have an automated app-release pipeline that we can be proud of.

## The Roadmap

* Part 1 - [App Store Connect API Key for authentication](/posts/migrate-to-fastlane-1-api-key/)
* Part 2 - Storing certificates/profiles in a GCP Bucket
* Part 3 - Reusing existing certificates, profiles and keys
* Part 4 - Migrating to manual app signing
* Part 5 - Provisioning Profile device list
* Part 6 - Uploading to TestFlight
* Part 7 - Reducing secrets sprawl

---

# Part 1 - App Store Connect Key

We'll start off with the lowest hanging fruit: switching our API authentication to using a private key.

Any operation performed by Fastlane that involves the App Store (eg. submitting to TestFlight, releasing the app) or certificates/profiles (eg. downloading, updating, etc) requires authentication to prove that you are allowed to do so.

## Credentials: A World of Pain

If you've been releasing your app manually, you'll be familiar with logging-in to App Store Connect with your email address and password in the browser.

Fastlane emulates that (cookie-based web session) authentication by default, using an unofficial version of the App Store Connect API. Performance and stability reasons aside, it is __not recommended due to enforced Two Factor Authentication__. In other words, the app owner's account must have (Apple's) 2FA enabled. If you are not said person and need to perform an app release, you will need to pester them often for a generated 2FA code. 

For the same reason, CI/CD will not be possible using this method as we can't (feasibly) input the ever-changing 2FA code.

Frustrating? Yes. Let's fix that first for our sanity.

## Private Key: The Better Way

The recommended approach is to use the official [App Store Connect JSON API](https://developer.apple.com/app-store-connect/api/), and authenticate using a JSON Web Token (JWT). All of that is taken care of by Fastlane under the hood, provided that we pass it a private key that can be used to generate said token.

More importantly, by using the official API we will not be prompted for a 2FA code.

### Generating the Key

A token can be generated on App Store Connect.


WIP


> ⚠️ Your account must have the __Admin__ role to generate a token (or even see a link to the page). The __App Owner__ role is not sufficient.

> 🤯 App Store Connect occasionally has weird caching issues. If your role was just updated to Admin but you still don't see the tab to create the token, try logout of App Store Connect and restart the browser.

Let's download the `.p8` key file. Assuming your Fastlane files are stored in a `fastlane` folder, we should place the key in the parent of this folder.

```
app/
├── app_store_connect_api_key.p8
└── fastlane/
    └── Fastfile
```

__Do not commit the key to your repository__, and be sure to add it to your `.gitignore` file.

As with any private key or token, you must take care who has access to it, as it grants the holder admin permissions to the API.

### Using the Key

From there it's a matter of calling the `app_store_connect_api_key` action in your `Fastfile`.

```ruby {linenos=table,hl_lines="2-6"}
lane :beta do |options|
  app_store_connect_api_key(
    key_id: ENV["ASC_API_KEY_ID"],
    issuer_id: ENV["ASC_API_ISSUER_ID"],
    key_filepath: "app_store_connect_api_key.p8"
  )
  upload_to_testflight # This action needs the token
end
```


Anything that requires interaction with App Store Connect (eg. TestFlight, registering devices with match, generating certs with match), will require a call to the `app_store_connect_api_key` action first.

The action will return a dictionary/hash map containing the JWT Token. Technically we could pass the returned value as a parameter to each action that requires the token, but I find that totally unnecessary. The action automatically puts the key into the context, so **you don’t need to do anything further**.

There's many approaches to how you can do this. Personally, I prefer to create a lane for injecting the key and switch to that lane when another lane requires the key.

```ruby {linenos=table,hl_lines="10"}
lane :setup_app_store_connect_key do |options|
  app_store_connect_api_key(
    key_id: ENV["ASC_API_KEY_ID"],
    issuer_id: ENV["ASC_API_ISSUER_ID"],
    key_filepath: "app_store_connect_api_key.p8"
  )
end

lane :beta do |options|
  setup_app_store_connect_key
  upload_to_testflight
end
```

Great! We're now using the key for authentication when running Fastlane locally.

## CI/CD

In CI/CD, we won’t have the `p8` file since it’s not pushed to the repository, and assuming you're using GitHub Actions, we cannot upload the key file. 

The solution is to store the contents of the key as a GitHub secret, and then inject it.

```ruby {linenos=table,hl_lines="4"}
app_store_connect_api_key(
  key_id: ENV["ASC_API_KEY_ID"],
  issuer_id: ENV["ASC_API_ISSUER_ID"],
  key_content: ENV["ASC_API_KEY_CONTENT"]
)
```

```yaml {linenos=table,hl_lines="4-6"}
- name: Register new devices
  run: fastlane ios register_new_devices
  env:
    ASC_API_KEY_ID: ${{ secrets.ASC_API_KEY_ID }}
    ASC_API_ISSUER_ID: ${{ secrets.ASC_API_ISSUER_ID }}
    ASC_API_KEY_CONTENT: ${{ secrets.ASC_API_KEY_CONTENT }}
```

That being said, we don't always want to have to inject the key contents. When running Fastlane on your own development machine you'll have the key as a file rather than as an environment variable.

Luckily, we can make use of the in-built `is_ci` action to check if we’re in a CI/CD environment.

```ruby {linenos=table,hl_lines="4"}
app_store_connect_api_key(
  key_id: ENV["ASC_API_KEY_ID"],
  issuer_id: ENV["ASC_API_ISSUER_ID"],
  key_content: is_ci ? ENV["ASC_API_KEY_CONTENT"] : "app_store_connect_api_key.p8"
)
```