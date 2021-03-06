---
layout: post
title: Diving a bit deep into Bitrise, jarsign, zipalign, apksigner and down the rabbit holes
permalink: diving-a-bit-deep-into-Bitrise,-jarsign,-zipalign,-apksigner-and-down-the-rabbit-holes
---

### TLDR

*   The combination of update of Android Gradle Plugin to [3.6.1](https://developer.android.com/studio/releases/gradle-plugin#3-6-0) and singing the APK with `Jarsigner` causes the APK to fail with error ```"Failure [INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2]" ```
*   One solution as mentioned in the release notes of the plugin update is to opt-out of the new default settings.
*   Second solution is to sign the APK with Gradle instead of "Android Sign Step" from Bitrise.

One fine afternoon and I hear the slack sound of a new message. Someone was unable to install one of the latest APKs from our #stable-build channel. I asked if they are trying to install the first time to understand the potential error surface. I asked the question, "Did the previous versions use to work for you?" and the reply was yes and they sent me a link to the previous working version, which was very helpful. Thanks!



### Check the intensity of 🔥

Naturally, I tried to reproduce the issue by installing the APK on my device, while fully hoping it works.... "**it didn't !**". It failed with the error. 

> Failure [INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2]

(mild panic, as we released the same build recently, time to head to Playstore to check if we mega screwed up). Things via Playstore were all fine, APK size was almost the same as before and it was installable.

### Following the breadcrumbs

Till now I was building the APK via Bitrise to reproduce the issue. I started building APK from my laptop. Because waiting on Bitrise for half an hour for each commit wasn't that nice. 

APK built from Gradle or AS was working. We did test the QA APK before releasing it. We built the APK on a local machine instead of Bitrise. Continuing the detective work, going through many commits, I found out that _one_ commit. It had some android libraries updates like constraint-layout and material-design but also Android-Gradle-Plugin version to [3.6.1](https://developer.android.com/studio/releases/gradle-plugin#3-6-0).

I thought must be the libs update to newer versions so let's check the Gradle-Plugin update first. and be sure that it is not the problem. But surprise! After the only plugin update, it increases our APK size, hmm, that's not expected. Well ok, let's check release notes of the changes, [lo and behold!](https://developer.android.com/studio/releases/gradle-plugin#extractNativeLibs)

> Native libraries packaged uncompressed by default When you build your app, the plugin now sets extractNativeLibs to "false" by default. That is, your native libraries are page aligned and packaged uncompressed. While this results in a larger upload size, your users benefit from the following: Smaller app install size because the platform can access the native libraries directly from the installed APK, without creating a copy of the libraries. Smaller download size because Play Store compression is typically better when you include uncompressed native libraries in your APK or Android App Bundle. If you want the Android Gradle plugin to instead package compressed native libraries, include the following in your app's manifest:

    <application
        android:extractNativeLibs="true"... >
    </application>

It was normal behaviour to have the APK size increased, but why it isn't installable? To be precise, APK from Bitrise isn't installable but building from local machine is fine. Let's find out how Bitrise is generating the APK? Why that APK works via Playstore but doesn't when we try to install it on our devices?

Meanwhile, to let internal people install our APK via Bitrise QRcode. I created the PR to set exactly above mentioned setting for Native libraries. With that change, APK size was back to previous size, and APK from Bitrise worked. Phew, now we have some breathing room to explore, and internals can still install and test our APK like they used to.

### Let the rabbit hole begin

So I head over to the Bitrise and check how we build the APK. I noticed we build and sign twice. One for the Playstore and one for internal downloads. That might be the reason why it's working via Playstore. Ah we upload an App-Bundle to the Playstore and build a normal release-signed APK for our internal usage. Then Playstore optimizes the App-Bundle and the actual download for our users is still almost the same as before, nice!

Digging deep into how Bitrise is building the APK I see it is nothing out of usual. Calling the same assembleRelease Gradle task. Bitrise is also signing the APK, and in my local tests, I only worked with debug signing keys and not the actual credentials used by Bitrise. I downloaded the signing keys and passwords from Bitrise and generated the APK and signed it via Android Studio. It worked fine 🙃.

In Android Studio (AS), to sign the APK there are two options. Sign the APK with signature 1 or 2 and both 🤷‍. I tried all three variants and APK was still installable from all variants. Unable to reproduce via AS, I thought let's check how the signing is being done on the CI. Turns out for signing, Bitrise provides an Android-Sign Step which is using the approach to run Jarsign on the APK first and then Zipalign it. I tried same steps via command line on my machine and finally, the errors showed up locally as well.

Ok as next step I set the "extractNativeLibs to true" locally as suggested in the release notes of the android-gradle-plugin and re-ran the same steps and the APK was installable again. Now more curious than ever I head over to read through docs about how to sign the Android APK, and I saw this there is new APKSign utility also available instead of using JARSign.

The only change mentioned in the doc is that if you use JARSign you need to Zipalign the APK _afterward_ as after the JARSign the 4-byte alignment is broken and you need Zipalign tool to align with 4-bytes again.

#### Steps to sign the APK and verify the alignment afterwards

1. Sign the APK with jarsigner
```bash
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore $APP_KEY app-release-unsigned.apk alias -storepass pass
```
2. Align the APK and page align the shared objects as well. ([source](https://developer.android.com/studio/command-line/zipalign))
```bash
zipalign -v -p 4 app-release-unsigned.apk aligned.apk
```

    (**-v** is for alignment, from the docs "The <alignment> is an integer that defines the byte-alignment boundaries. This must always be 4 (which provides 32-bit alignment) or else it effectively does nothing." ), you can read more about zipalign [here](https://developer.android.com/studio/command-line/zipalign).
3. Check the alignment is correct.
```bash
zipalign -c -v -p 4 aligned.apk
```
    
    **-c** to check the alignment.

But when I ran the check after printing a bunch of (OK - compressed) after file names, in the end, it said: "Verification FAILED". My assumption was with the uncompressed native libraries zipalign is not aligning those files correctly, so it could be a bug in zipalign, but then I remember there is something like apksigner and in the docs, it mentions you should sign the APK with apksigner now and the App-Bundles with Jarsigner.

I tried the steps mentioned in the docs. One thing which was different from previous steps is you are supposed to zipalign the APK first and then sign in with apksigner as the signature of the APK will be invalid if the file contents are changed after the signing (makes sense).

From the docs of Zipalign:

> **Caution:** You must use zipalign at one of two specific points in the app-building process, depending on which app-signing tool you use:

> If you use [**apksigner**](https://developer.android.com/studio/command-line/apksigner), zipalign must only be performed **before** the APK file has been signed. If you sign your APK using apksigner and make further changes to the APK, its signature is invalidated.

> If you use [**jarsigner**](https://docs.oracle.com/javase/tutorial/deployment/jar/signing.html), zipalign must only be performed **after** the APK file has been signed.

I used the `zipalign` _before_ signing the APK with `apksigner`. I checked the alignment with the `-c` option but it still said verification failed at the end. I proceeded with the `apksinger` and signed the APK but it still failed with same error. #sadpanda.

### What worked

I generated the APK with assembledRelease again, and this time I ran the zipalign -c check before aligning the APK. It said "Verification successful"! The APK generated via assembleRelease Gradle task is already aligned or it would seem so. I signed the APK with apksigner and viola! the APK with uncompressed native libraries is installable again.

### Related links

1.  [https://developer.android.com/studio/build/building-cmdline](https://developer.android.com/studio/build/building-cmdline)
2.  [https://developer.android.com/studio/command-line/zipalign](https://developer.android.com/studio/command-line/zipalign)
3.  [https://developer.android.com/studio/command-line/apksigner](https://developer.android.com/studio/command-line/apksigner)
4.  [https://medium.com/androiddevelopers/smallerapk-part-8-native-libraries-open-from-apk-fc22713861ff](https://medium.com/androiddevelopers/smallerapk-part-8-native-libraries-open-from-apk-fc22713861ff)
5.  [https://developer.android.com/studio/releases/gradle-plugin#3-6-0](https://developer.android.com/studio/releases/gradle-plugin#3-6-0)
6.  [https://devcenter.bitrise.io/code-signing/android-code-signing/android-code-signing-using-bitrise-sign-apk-step/](https://devcenter.bitrise.io/code-signing/android-code-signing/android-code-signing-using-bitrise-sign-apk-step/)
7.  [https://github.com/bitrise-steplib/steps-sign-apk](https://github.com/bitrise-steplib/steps-sign-apk)
