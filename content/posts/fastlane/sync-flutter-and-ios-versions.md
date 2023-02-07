---
title: "Force-sync your Flutter and iOS versions"
date: 2023-01-10T11:23:00+09:00
tags: [ios, fastlane, flutter]
categories: [ios, fastlane, flutter]
draft: false
---

While iOS app distribution can already be a challenge on its own, Flutter is a black box wrapping around that to add an extra level of _undocumented pain_.

Just like regular iOS development, we can apply Fastlane to automate our app distribution pipeline, but there's plenty of unexpected hiccups that happen along the way. Flutter being a cross-platform development solution however, it tries to abstract away differences in the platforms at the cost of some black magic.

One particularly annoying issue I faced is that the version I defined for my Flutter app was often different to what the iOS app claimed it to be.

## The Source of Truth

As a quick recap on how Flutter's versioning works, we define our Flutter app's version in the `pubspec.yaml` file.

It expects [semantic versioning](https://semver.org/), and a build number.

```dart
version: 1.2.3+4

// Major = 1
// Minor = 2
// Patch = 3
// Build = 4
```

The value defined here is automatically used by the iOS and Android app built for us by Flutter.

For iOS, this becomes like so:

|`Info.plist` Raw Key|Description|Value Based on Above Example|
|--|--|--|
|`CFBundleShortVersionString`|Version string|`1.2.3`|
|`CFBundleVersion`|Build number|`4`|

* The version string (`major`.`minor`.`patch`) is the public-facing version. People can see this on the App Store.
* The build number on the other hand is mainly for distribution platforms to differentiate between multiple copies of your app that have the same version string. 
  * For example, it's very common in large teams to have a pipeline that automatically distributes your app to developers/internal testers whenever a commit is pushed to the main branch. In such a system, developers push many changes under the same version string before a new version is released. Each change automatically increments the build number so that the uploaded version is unique in the eyes of the distribution platform.
  * This is important to note because some distribution platforms like [TestFlight](https://developer.apple.com/testflight/) will not allow you to upload your app if the version string & build number combination has already been uploaded for your app identifier.

In Flutter land, `pubspec.yaml` is our source of truth in the sense that each platform (iOS, Android) will receive the version defined in that file when building the app, as opposed to each platform defining the versions themselves and duplicating these values all over the place.

## How does iOS get the Flutter version?

Flutter generates two files for us in the `ios/Flutter` folder:

1. `flutter_export_environment.sh`
2. `Generated.xcconfig`

These are essentially a Key-Value map of build settings - two of which are of particular interest:
```bash
// Generated.xcconfig
FLUTTER_BUILD_NAME=1.2.3
FLUTTER_BUILD_VERSION=4

// flutter_export_environment.sh
export "FLUTTER_BUILD_NAME=1.2.3"
export "FLUTTER_BUILD_VERSION=4"
```

Flutter takes the version in `pubspec.yaml` and generates these two files with the version string (amongst other things). 

`Generated.xcconfig` is an _Xcode build configuration file_ that is read during compile-time when building the iOS app. These build settings are loaded into the project during compilation as if they had been declared in the `xcodeproj`.

> Check out [here](https://nshipster.com/xcconfig/) if you're interested to see more on what `xcconfig` files can do

As mentioned before, the version string is read from the `CFBundleShortVersionString` and `CFBundleVersion` keys inside `Info.plist` and bundled with the app. Unlike a conventional iOS app however, these point to the value received from `Generated.xcconfig` rather than a hardcoded value.

Quite a lot of misdirection, but at least it gets the job done.

... usually.

## So what's the problem?

There's two ways to compile/run the iOS app:

1. Run the `flutter build` command in CLI
2. OR open Xcode and compile the app

Regardless of which way you choose, the versions will still be pulled out of `Generated.xcconfig` at compile-time.

Great! ... but actually it turns out that the two files (`flutter_export_environment.sh` and `Generated.xcconfig`) are __only generated when using the `flutter build` command__. And therein lies the problem with the Flutter black box.

In other words, if you change the version in `pubspec.yaml` and then compile the app with Xcode, the bundled version will still be the old version because this process is not managed by Flutter.

It's a fairly common scenario and I'm very surprised there isn't any ruckus about it. In the past I've accidentally pushed an app labelled with the wrong version to TestFlight before due to this issue, and there's no way to delete it other than to "expire" the build. Annoying.

# The Solution

I must preface this by saying this is a hacky workaround. We're tampering with generated files, and that always bears the risk of breaking in a future update of Flutter. That being said, this solution works for me and the trade-off is worth the risk. YMMV.

Basically, we will want to overwrite the version values written in the two generated files (`flutter_export_environment.sh` and `Generated.xcconfig`) before compiling the iOS app.

There's many ways to achieve this: a bash script, a git hook, a run script (for your xcodeproj) and more. Since I'm already using Fastlane to automate app distribution, I decided to write a [custom Fastlane action](https://docs.fastlane.tools/create-action/).

## Custom Fastlane Action

Create a new Fastlane action using the `fastlane new_action` command, and give it a name. For our intents and purposes, I named it `sync_ios_with_pubspec_version`.

Next we'll write a Ruby script to read the version from the `pubspec.yaml` file, and overwrite the values in the two generated files.

```ruby
require 'yaml';

module Fastlane
  module Actions
    class SyncIosWithPubspecVersionAction < Action
      def self.run(params)
        yaml = YAML.load_file("pubspec.yaml")
        version = yaml["version"]
        
        matches = /^(\d+)\.(\d+)\.(\d+)\+(\d+)$/.match(version).captures

        major = matches[0]
        minor = matches[1]
        patch = matches[2]
        build = matches[3]

        pubspsec_version = "#{major}.#{minor}.#{patch}"

        [
          "ios/Flutter/flutter_export_environment.sh",
          "ios/Flutter/Generated.xcconfig",
        ]
        .each do |file_name|
          edited_data = File.open(file_name) do |f|
            f.read
              .gsub(/FLUTTER_BUILD_NAME=[\d\.]+/, "FLUTTER_BUILD_NAME=#{pubspsec_version}")
              .gsub(/FLUTTER_BUILD_NUMBER=\d+/, "FLUTTER_BUILD_NUMBER=#{build}")
          end
          IO.write(file_name, edited_data)
        end
      end

      def self.description
        "Forces the iOS app's version to be overwritten with the value in pubspec.yaml"
      end

      def self.is_supported?(platform)
        platform == :ios
      end
    end
  end
end
```

Finally, we just need to invoke our new custom action from any `Fastfile` lane that builds the app. This will ensure the version is always synced.

```ruby {linenos=table,hl_lines="2"}
lane :beta do |options|
    sync_ios_with_pubspec_version
    build_app(...)
    upload_to_testflight(...)
end
```

Not exactly elegant, but at least the problem is solved for now.