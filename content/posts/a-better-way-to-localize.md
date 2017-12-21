---
title: "A Better Way to Localize"
description: "Localization is hard. Especially if you don't plan for it ahead of time."
author: "njooma"
date: 2017-12-01T21:38:57-05:00
tags: []
draft: true
---

Localization is hard. Especially if you don't plan for it ahead of time.
<!--more-->

Localization on iOS is kind of a mess. You have a `Localizable.strings` file that contains all your strings. But if you need to handle plurals (e.g. "1 day remaining" vs "2 days remaining"), you need a `Localizable.stringsdict` file, which is written in XML (always fun). On top of that, all the string IDs are stringly-typed (which, in a strongly-typed language like Swift, feels out of place), you can't autocomplete string IDs, there's no easy way to find unused strings, etc., etc. 

Something Android does extremely well is forcing you to prepare your app for localizations. Android Studio warns you when you've hard-coded strings and gives you an easy way to turn those hard-coded strings into a localized `key: value` in their version of a strings file. Literally just a click of a button. Android Studio also autocompletes these string IDs for you when you try to access them in your code. (I would be remiss if, while showering praises on Android, I didn't mention that the entire Android strings file is XML...)

Xcode does not. It doesn't yell at you when you haven't localized a hard-coded string, it won't let you autocomplete string IDs, nor will it tell you when you have unused string IDs, forcing you to clean up bloated strings files by hand.

But you really should localize your app. Or at least make it so that you won't regret it if you have to localize later. But what's incredibly annoying is having to use `NSLocalizedString(_:comment:)` everywhere! Seriously. It takes up a lot of space every single time you want to set text. And if you want to set the title of a button, you have to nest this call inside of `setTitle(_:for:)`. And god forbid you have any string formatting to do, or else you might get a line of code that looks like this:

{{< highlight swift >}}
// "stringWithFormatting" = "A string with 2 variables: %1$@ and %2$@";
button.setTitle(
    String(format: NSLocalizedString("stringWithFormatting", comment: "A localized string"), 
        variable1, variable2), 
    for: .normal)
{{< / highlight >}}

#### Yeah. ####

Of course you could split it up into multiple lines and create objects to house intermediary values, but my point is that it shouldn't be this bad.

So how can we make this better?

Well, if you picked up on my foreshadowing from the top of this post, you'll know that one of the bits of Android localization I like most is autocomplete of string identifiers. And one of the things I hate most about iOS localization is maintaining a strings file in a constantly changing app. 

The solution I came up with is organizing your localization string IDs in a single file using `enum`s. Probably not the most original of ideas, but I like it. This solves a bunch of our problems: you can easily see which strings are not called anywhere in your code, you get autocomplete, and BONUS: you don't have to do crazy-ugly-super-long-basically-run-on-sentence lines of code.

{{< highlight swift >}}
protocol LocalizationKey {
    var localized: String { get }
}

extension LocalizationKey where Self: RawRepresentable, Self.RawValue == String {
    var localized: String {
        return NSLocalizedString(self.rawValue, comment: "")
    }
}

enum LocalizationKeys: String, LocalizationKey {
    // "stringWithFormatting" = "A string with 2 variables: %1$@ and %2$@";
    case stringWith2Variables = "stringWithFormatting"
}

button.setTitle(
    String(format: LocalizationKeys.stringWith2Variables.localized, variable1, variable2), 
    for: .normal)
{{< / highlight >}}

You can even namespace your keys using nested `enum`s, but you lose one benefit if you do that: Xcode will warn you if sibling `case`s have the same value, since you cannot have multiple `case`s with the same value within the same `enum`. By nesting `enum`s, you will no longer be warned of potential duplicate strings.

{{< highlight swift >}}
enum LocalizationKeys: String, LocalizationKey {
    case localizedIDOne = "localizedString"
    case localizedIDTwo = "localizedString" // Will give you an error for duplicate value

    enum NameSpace: String, LocalizationKey {
        case localizedIDThree = "localizedString" // Will NOT give an error for duplicate value
    }
}
{{< / highlight >}}

There are some obvious downsides with this method: you cannot use it in storyboards, and you lose the ability to automatically export localization files using Xcode's built-in "Export for Localization" option.

To address the first: if you haven't figured it out yet, you will soon, but storyboards kinda suck. Using them as part of a team can get extremely frustrating, it's nearly impossible to handle any merge conflicts, and I usually end up setting a lot of content in code anyway (less and less because Apple keeps making strides, like named color assets, but storyboards still have some work to do). I guess if you're living the storyboard life you theoretically could create an extension on `UILabel` that allowed you set the `LocalizationKey` via `@IBInspectable`. Or by subclassing `UILabel`. So not impossible to use with storyboards, but you lose some main benefits (namely strongly-typed keys).

To address the second: It's a little more work up-front, but you will have to create your own `Localizable.strings` file (and `.stringsdict` if needed). Every translation service I have used will allow you to upload a `.strings` file and can parse and translate your phrases from there. So a mild up-front inconvenience in exchange for larger benefits. And I'm sure if you really wanted to automate, you could script the creation of a `.strings` file by extending the `LocalizationKey` protocol with `comment: String` and `defaultValue: String` fields and then iterating over the `case`s. 

So yes, there are drawbacks. But like I said in the opening of this post, localization for iOS not great and I found that this makes it more tolerable.

<iframe src="https://open.spotify.com/embed/track/3RHxEG6JqPKNesUpEv3FUg?theme=white&view=list" width="100%" height="380" frameborder="0" allowtransparency="true"></iframe>