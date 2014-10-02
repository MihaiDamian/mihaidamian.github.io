---
layout: post
title: "Managing iOS and Android localized strings"
date: 2014-10-02 22:27:40 +0300
comments: true
categories: 
- iOS
tags:
- Android
- localization
- internationalization
---

If you are developing an app for iOS and Android, and if you need to translate the app in a foreign language, you will be faced with the decision of translating the user-facing strings in the two apps independently or together. Translating the apps independently is the easier option as far as development effort is concerned. On the other hand, you can take advantage of the fact that roughly the same strings will be used on the iOS and Android versions and save costs on the translation side. Translating the apps together comes with some difficulties, as we'll see in this article.

Before we dive deeper, I'll just want to note that at this point you should already be familiar with string internationalization on iOS and Android. If not, I recommend you start by reading the iOS [Internationalization and Localization Guide](https://developer.apple.com/library/prerelease/ios/documentation/MacOSX/Conceptual/BPInternational/Introduction/Introduction.html) and [Supporting Different Languages](http://developer.android.com/training/basics/supporting-devices/languages.html) for Android. There are of course other resources except strings that need to be considered when localizing an application, but this article will focus only on strings.

### Making tradeoffs

On the iOS side, Xcode 6 came with several improvements to its internationalization process. *genstrings* and *ibtool* no longer have to be used to extract strings from code and interface builder files. Instead, this is now achieved in a single step by exporting to XLIFF. *genstrings* and *ibtool* were particularly annoying to use as they were not capable of merging the generated strings files with previous outputs. The new export/import feature solves this problem by extracting the localizable strings to an external XLIFF file that is not directly used by the app and importing and merging localized strings from XLIFF files into strings files.

Other problems, however, remain unsolved: base internationalization offers no way of adding comments and all strings, except those used by base internationalization, are indexed by the string itself, in the projects' development language. The last point is occasionally problematic if you have a string that's used as a homonym in the development language. Take for example the English word *share* that can be used as the verb *to share* or as a noun as in a stock share. In another language you will likely need two different words for the two meanings so in this case it's not possible to use a single string key.

But for cross-platform projects, the biggest issue is that iOS localizable strings files generated as part of base internationalization, or even the XLIFF exported output, are not readily usable on Android (and vice versa). Base internationalization string files contain key-value pairs where strings are indexed by Interface Builder object ids:

``` bash
"CJ9-Yv-rto.text" = "Some text";
```

This type of ids are practically meaningless in an Android project and you definitely don't want to write code that uses them as resource names. For this reason, base internationalization can't be used if we hope to use a single set of localizable strings on iOS and Android.

### A typical internationalization-localization workflow

Let's briefly look over the typical activities performed when internationalizing and localizing an app developed for iOS and Android. 

During development localizable strings are used instead of hardcoded strings. This means that application code loads user-facing strings based on pre-determined keys from localized string files. Initially no translated strings are available so the only locale used is that of the projects' development language.

At some point, all localizable strings, from both platforms, need to be gathered and sent to translators. Commonly the two platforms will use roughly the same set of strings. To send out a single file containing all strings we need to come up with a common format and merge the iOS and Android strings into it. This can be either one of the platform native formats or an entirely different one. Either way we'll need to take into account that string formatting characters like %@ and %s will be different for the two platforms. While translators are busy doing their work, application development continues and new strings are added while others may be deleted or modified.

When the translated strings are received back they need to be merged with the new, untranslated strings. They also need to be converted back in platform specific formats (strings files for iOS, XML for Android). Sometime in the future, the remaining untranslated strings need to be sent again to the translators.

These steps are often repeated multiple times. Crowdsourced translation services make translating small and frequent batches of strings easy. Doing this will help ironing out any issues related to translation well before the release date. Naturally, when the application is ready for release all strings should be translated.

For medium to large projects there will be a significant number of strings to work with. Some of the steps, like the merging into and out of the common strings format, will be quite labor intensive. Automating the process is definitely a necessity. Fortunately there is an open-source command line tool called [twine](https://github.com/mobiata/twine) that can help us achieve this.

### Preparing you localizable resources for use with twine

Like XLIFF export/import in Xcode, Twine works on a single master file that holds all your localizable strings with translations in the available locales. To import the existing strings from the iOS and Android projects start by creating an empty text file:

``` bash
touch strings.txt
```

Next, import localizable strings from the iOS project:

``` bash
twine consume-all-string-files strings.txt PathToIOSProject --developer-language en --consume-all --consume-comments
```

Now do the same for the localizable strings in the Android project:

``` bash
twine consume-all-string-files strings.txt PathToAndroidProject --developer-language en --consume-all --consume-comments
```

This will merge the strings from the two projects, by string keys, in a common master file. If you were already using the same string keys on both iOS and Android you'll end up with no duplicates. The command also identifies all the available locales based on project folder structure and will import any existing translations as well.

Notice that when native strings are consumed, string formatting special characters, like %s, will be changed to a common format. Also notice that *consume-all-string-files* will search through all iOS strings files and all Android XML resource files. While this works great for the initial creation of the master file, in practice you'll save yourself a lot of effort if you work with a single native strings file, per platform, per locale.

### The workflow with twine

Let's look again at the typical internationalization and localization actions, this time with twine helping us along the way.

#### Adding, modifying and deleting strings

When you need to add a new localizable string during development, simply add it to the master file. In other words, never manually edit the platform specific strings files. These files will be instead generated automatically as we'll see next. Modifying and deleting strings is a bit problematic, since such changes can only be performed when a single platform is impacted. You can't for instance delete a string if the iOS version of the app no longer needs it, since it may still be in use on Android.

#### Updating iOS and Android localization files from the master file

Updating the localizable strings files is as simple as this:

``` bash
twine generate-all-string-files strings.txt PathToProject
```

Again, twine will use project folder structure to determine the locales the project is using and will update the necessary files for all the locales. It is recommended to include this command in your build process so that if the master file is modified the localization files can be updated with a simple build command. In Xcode you can to this by adding a custom Run Script build phase. Just make sure you order this build phase to be executed before the Copy Bundle Resources phase.

#### Sending out strings to the translation team

When you are ready to send a batch of strings out for translation, you can send the single master file. The twine master file uses a human-friendly, easy to read syntax, inspired by Windows [INI](http://en.wikipedia.org/wiki/INI_file) files. Translators should have no problem filling in the missing translations. Optionally, you could export to [Gettext PO](http://en.wikipedia.org/wiki/Gettext) files if you need to provide you translation team with a more commonly used file format.

#### Merging back updates from the translation team

Merging back translated strings works similarly with importing strings from platform specific strings files. Here's an example that merges a file received from translators (localized.po) with the master file (strings.txt).

``` bash
twine consume-string-file strings.txt localized.po --lang es
```

Finally, all that remains is to update the iOS and Android localizable strings files as we've seen before.

### Conclusions

In the end, we have to wonder if it's really worth translating the iOS and Android versions together. Twine helps automate many of the labor intensive parts of the process but coordination between developers building the iOS and Android apps is still needed, especially when modifying and deleting existing strings. Translation effort is certainly reduced, but this has not come for free. The is no clear-cut answer and everything depends on the particularities of your situation. The factors that will swing the balance are the number of languages that will have to be supported, team composition (will the same developers work on both platforms or not?), number of localizable strings, feature overlap and the relative cost of development versus translation.
