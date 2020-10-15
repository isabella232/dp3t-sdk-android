# DP3T-SDK without proprietary dependencies

## Purpose of this fork

Currently, many contact-tracing applications rely on the Google-Apple Exposure
Notification framework, known as GAEN. On Android, GAEN is part of the Google
Play services, in order to integrate with the OS at a low level. As a
consequence, if a phone runs an OS that does not have Google Play services
installed, contact-tracing applications will not work. This is the case for
example for:

- Commercial OSes that do not (or cannot) come with Google Play services
  installed (e.g. Huawei).
- Open Source OSes (e.g. LineageOS).

The purpose of this fork is to remedy the situation by creating a version of
the DP3T-SDK that does not depend on the GAEN libraries for Exposure
Notification.

The primary user of this library is the SwissCovid application, but other
applications that use the DP3T-SDK should be easily adaptable, and the approach
is applicable as well to applications that do not use the DP3T-SDK but
perhaps access the GAEN libraries directly.

## Approach

After examining various possibilities, we have settled to base our approach on
the [microG](https://microg.org/) project. Indeed, microG aims to provide a
free re-implementation of the complete Google Play services, in a way that
makes it completely transparent to the applications using them. In order to
achieve this, microG uses some clever tricks which make its installation
non-trivial for end-users. Furthermore, for our project, we only need the
subset of microG that re-implements the GAEN functionality.

Consequently, we are reusing the relevant parts of microG related to GAEN, but
bundling them as libraries with the app instead of having them installed as a
separate set of services. This keeps the goal of requiring very minimal
modifications on the final application: indeed, the source code remains the
same since the GAEN API is preserved, and only build-related changes are needed
to ensure that the GAEN calls are handled by the libraries instead of the
Google Play services.

## Implementation

These are the changes that were done to the original DP3T-SDK in order to use
microG as libraries:

- Modify the build files (`build.gradle`) replacing the dependencies on Google
  Play services with dependencies on the relevant microG modules:

  - play-services-base
  - play-services-base-api
  - play-services-base-core
  - play-services-base-core-ui
  - play-services-basement
  - play-services-tasks
  - play-services-nearby
  - play-services-nearby-api
  - play-services-nearby-core
  - play-services-nearby-core-ui
  - play-services-nearby-core-proto

The changes required on the Calibration app in order to use the modified SDK are:

- Modify the build files (`build.gradle`) adding the dependencies on some
  microG modules.

And finally, the changes required on the SwissCovid app in order to use the
modified SDK are:

- Modify the build files (`build.gradle`) to take the SDK library locally
  instead of the official one from Maven, as well as to add the relevant microG
  libraries.

- Modify the UI to differentiate the app from the official version
  (in particular, renaming to LibreCHovid) and notify the user.

- A few minor tweaks (not clear whether they are related to the change).

## Build

Automatically built releases of the SDK and LibreCHovid app are available on
GitHub, respectively at:

- https://github.com/c4dt/dp3t-sdk-android/releases
- https://github.com/c4dt/dp3t-app-android-ch/releases

In order to build the SDK and the LibreCHovid application manually, follow these
steps (these are the same as the workflows for GitHub actions):

- Install the latest version of the Android SDK (FIXME: or is the JDK only needed?).

- Checkout the `microg-nearby` branch of the SDK and LibreCHovid app forks:

```
$ git clone -b microg-nearby git@github.com:c4dt/dp3t-sdk-android.git
$ git clone -b microg-nearby git@github.com:c4dt/dp3t-app-android-ch.git
```

- Build the SDK:

```
$ cd dp3t-sdk-android/dp3t-sdk
$ ./gradlew assembleRelease
$ cd ..
```

- Build the app:

```
$ cd ../dp3t-app-android-ch

# Copy the SDK library
$ rm -f ./app/libs/*
$ cp ../dp3t-sdk-android/dp3t-sdk/sdk/build/outputs/aar/sdk-production-release.aar ./app/libs/

# For the following step, you need to use your keystore file location and
# passwords, or create a new one. Further information is available at
# https://developer.android.com/studio/publish/app-signing .
# Otherwise, you can use the `assembleProdDebug` target instead to build a
# debug app.
$ ./gradlew assembleProdRelease -PkeystoreFile=<keystoreFile> -PkeystorePassword=<keystorePassword> -PkeyAliasPassword=<keyAliasPassword>
```

If all proceeds without errors, the final APK can then be found at `./app/build/outputs/apk/prod/release/app-prod-release.apk`.

## Notes

### Debug

In order to verify the proper execution of the application, the following can be useful:

- Connect two (or more) phones to your development machine (ensure they are
  recognized by `adb devices`).
- Install the application on all the phones.
- Capture the logs of each phone:

```
$ adb -s <phone1_device> logcat --format color | tee -a phone1.logcat
$ adb -s <phone2_device> logcat --format color | tee -a phone2.logcat
...
```

- Monitor the exposure notification events between the phones:

```
$ tail -F *logcat | awk '/^==>/ {filename=$2; next} {print filename ":" $0}' | grep ExposureNotification:
```

### Building with modified microG libraries

The microG libraries are picked up from releases in the public Maven repository
during the build.
In order to use modified libraries, they must be published locally:

```
[GmsCore] $ ./gradlew publishToMavenLocal
```

The following build modifications can then be applied to use the local
libraries (adapt to current microG version).

For DP3T-SDK:

```
diff --git a/calibration-app/build.gradle b/calibration-app/build.gradle
index 18de7f7..0cb4bb5 100644
--- a/calibration-app/build.gradle
+++ b/calibration-app/build.gradle
@@ -11,7 +11,7 @@
 // Top-level build file where you can add configuration options common to all sub-projects/modules.

 buildscript {
-    ext.microgVersion = '0.2.17.204714'
+    ext.microgVersion = '0.2.17.204714-dirty'

     repositories {
         google()
@@ -25,6 +25,7 @@ buildscript {

 allprojects {
     repositories {
+        mavenLocal()
         google()
         jcenter()
     }
diff --git a/dp3t-sdk/build.gradle b/dp3t-sdk/build.gradle
index 0ff7675..c49604f 100644
--- a/dp3t-sdk/build.gradle
+++ b/dp3t-sdk/build.gradle
@@ -14,7 +14,7 @@ buildscript {
         classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.12'
 	}

-	ext.microgVersion = '0.2.17.204714'
+	ext.microgVersion = '0.2.17.204714-dirty'
 }

 allprojects {
diff --git a/dp3t-sdk/sdk/build.gradle b/dp3t-sdk/sdk/build.gradle
index 5abb45b..8daa0bd 100644
--- a/dp3t-sdk/sdk/build.gradle
+++ b/dp3t-sdk/sdk/build.gradle
@@ -160,5 +160,6 @@ dependencies {
 }

 repositories {
+	mavenLocal()
 	mavenCentral()
 }
```

For the SwissCovid app:

```
diff --git a/build.gradle b/build.gradle
index d6daf5f..7ba5d84 100644
--- a/build.gradle
+++ b/build.gradle
@@ -11,7 +11,7 @@
 // Top-level build file where you can add configuration options common to all sub-projects/modules.

 buildscript {
-	ext.microgVersion = '0.2.17.204714'
+	ext.microgVersion = '0.2.17.204714-dirty'

 	repositories {
 		google()
@@ -24,6 +24,7 @@ buildscript {

 allprojects {
 	repositories {
+		mavenLocal()
 		google()
 		jcenter()
 	}
```
