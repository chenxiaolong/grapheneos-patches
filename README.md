# My GrapheneOS patches

This repo contains a collection of patches that I apply to my personal GrapheneOS builds.

## Secure ADB for userdebug builds

Patches:
* [`0001-Unconditionally-enable-ro.adb.secure-1.patch`](./patches/build/soong/0001-Unconditionally-enable-ro.adb.secure-1.patch)

This enables `ro.adb.secure=1` unconditionally instead of only for `user` OS builds. This allows adb's normal authorization mechanism to work in `userdebug` builds.

## Adaptive brightness

Patches:
* [`0001-gmscompat-Add-support-for-Device-Health-Services.patch`](./patches/frameworks/base/0001-gmscompat-Add-support-for-Device-Health-Services.patch)

This allows using the Pixel OS' adaptive brightness implementation from the [Device Health Services](https://play.google.com/store/apps/details?id=com.google.android.apps.turbo) package.

The adaptive brightness implementation does not override AOSP's automatic brightness implementation. Instead, it only adjusts the brightness curve parameter and tracks when the user manually adjusts the screen brightness to train the (offline) ML model. The ML training job runs periodically in the background (roughly every 10 hours if the device is charging and idle).

### Permissions

The patches allow Device Health Services to use the following permissions:

* `ACCESS_AMBIENT_LIGHT_STATS` (system)
* `BRIGHTNESS_SLIDER_USAGE` (system)
* `CONFIGURE_DISPLAY_BRIGHTNESS` (system)
* `RECEIVE_BOOT_COMPLETED` (user)

All other permissions are entirely removed.

**A couple notes:**
* This is not a perfect implementation because the system permissions are immediately granted when Device Health Services is installed. A proper implementation would ask for user consent like how GrapheneOS' Android Auto permission toggles work.
* Although the actual adaptive brightness functionality is entirely self-contained and does not depend on Google Play Services, the patches permit access to GMS Core and the Play Store. This is necessary to allow it to query feature flags that enable the use of the better `TFLiteSplineOfflineModel` (ML model ID 12) instead of the default `MonotonicGaussianOfflineModel` (ML model ID 6).

As of Android 15 QPR1, this is what the systems permissions grant access to:

#### `ACCESS_AMBIENT_LIGHT_STATS`

Allows access to ambient light sensor statistics collected for a week, split into one day intervals. The statistics for each day include the amount of time spent in an environments where the light level falls into a [set of predefined buckets](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-15.0.0_r13/services/core/java/com/android/server/display/AmbientBrightnessStatsTracker.java#60). Android records this data in `/data/system/ambient_brightness_stats.xml`.

#### `BRIGHTNESS_SLIDER_USAGE`

Allows access to [the last 100](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-15.0.0_r13/services/core/java/com/android/server/display/BrightnessTracker.java#101) brightness slider change events. Each event includes the following information from when the brightness slider was changed. ~~Struck-through~~ fields are available to, but ignored by Device Health Services.

* the selected brightness
* the timestamp
* ~~the app that is in the foreground~~
  * Only possible if the `PACKAGE_USAGE_STATS` permission is granted, which the patch unconditionally blocks
* ~~the Android user/profile that is making the change~~
* which physical display is having its brightness changed
* recent ambient light sensor values
* ~~the battery level~~
* the scale factor applied to the brightness due to the battery level
* ~~whether night mode is active and what color temperature it's configured with~~
* the strength of any active color filters
* the brightness prior to it being changed
* whether the current brightness curve is the default
* whether the current brightness curve includes a user-set brightness point
* ~~a histogram bucketing the V component of HSV for the entire screen's pixels~~
  * Android generates this data only if Device Health Services chooses to enable it

Android records this data in `/data/system/brightness_events.xml`.

#### `CONFIGURE_DISPLAY_BRIGHTNESS`

Allows access to query and modify the device's auto-brightness configuration, which contains the following properties:

* the brightness curve mapping lux (ambient light) to nits (screen brightness)
* a per-app brightness correction factor
* a per-app-category brightness correction factor
* parameters for AOSP's auto-brightness implementation's short term model
* whether color samples of the screen should be collected when tracking brightness slider events

Android persists this configuration in `/data/system/display-manager-state.xml`. It remains even if Device Health Services is disabled or uninstalled.

### Usage

 To use Device Health Services, first install the app. The APK will likely need to be extracted from the stock Pixel OS at `/product/priv-app/TurboPrebuilt/TurboPrebuilt.apk`. Even though the app is hosted in Google Play, it claims the app is incompatible unless the device is running the stock Pixel OS.

After it is installed, launch it once by going to Android's Settings -> Apps -> Device Health Services -> Storage & cache -> Manage storage. This is necessary because newly installed user apps (and force stopped apps) are not to allowed automatically run until the user manually opens it once, and there's no other easily accessible UI for Device Health Services.

Finally, reboot the device. Device Health Services will schedule the periodic ML model training job on boot.

## License

All patches are licensed under Apache 2.0, the same as the original code being patched. Please see [`LICENSE`](./LICENSE) for the full license text.
