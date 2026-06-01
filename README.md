# Treble Manifest — Lenovo Y700 (TB-9707F) LineageOS 23.2 GSI

Local manifest and extras for building / running an **unofficial LineageOS 23.2
(Android 16 QPR2) Treble GSI** on the **Lenovo Legion Y700 2022 (TB-9707F,
Snapdragon 870)**, based on [MisterZtr/LineageOS_gsi](https://github.com/MisterZtr/LineageOS_gsi).

On top of the upstream manifest this fork adds:

- **Face unlock (FaceSense) restoration** for the 23.2 line
- Ready-to-flash **Magisk modules** for two GSI quirks on this device
  (see [`modules/`](modules/))

---

## 1. Using the manifest

```bash
mkdir LineageOS && cd LineageOS
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs
git clone https://github.com/kkhunter2/treble_manifest.git .repo/local_manifests -b lineage-23.2
repo sync --force-sync --optimized-fetch --no-tags --no-clone-bundle --prune -j8
bash LineageOS_gsi/patches/apply-patches.sh .
```

Build (GAPPS, ext4 — matches the Y700 reference build):

```bash
. build/envsetup.sh
# treble_app must be pre-built once (Java 17): bash treble_app/build.sh
# NOTE: if breakfast/lunch mis-parses the combo, set the vars directly:
export TARGET_PRODUCT=lineage_arm64_bgN4 TARGET_RELEASE=bp4a TARGET_BUILD_VARIANT=userdebug
m systemimage -j"$(nproc)"
```

Flash `out/target/product/generic_arm64/system.img` with `fastboot flash system`.

---

## 2. Face unlock (FaceSense)

Face unlock was dropped when 23.2 rebased onto the RestlessOS patchset. It is
restored here via the **FaceSense** software provider (Paranoid Android "Sense"
/ Megvii engine) — camera-based, needs **no vendor face HAL**.

This manifest pulls the app + AIDL + engine:

```xml
<project path="packages/apps/FaceUnlock"
         name="Evolution-X/packages_apps_FaceUnlock"
         remote="git-hub" revision="bka" />
```

The matching framework patches live in the LineageOS_gsi fork. Upstream PRs:

- Framework + enablement: **MisterZtr/LineageOS_gsi#102**
- This manifest entry: **MisterZtr/treble_manifest#5**

Verified on TB-9707F: `dumpsys face` reports `provider: SenseProvider,
sensorId 1008`; `ro.face.sense_service=true` and
`feature:android.hardware.biometrics.face` are present; a face can be enrolled
and a successful authentication is logged.

---

## 3. Magisk modules ([`modules/`](modules/))

Runtime fixes for device quirks that don't belong in the generic GSI. Install
with the Magisk app (or `magisk --install-module <zip>`) and reboot. They live
in `/data/adb/modules`, so they **survive re-flashing `system.img`**.

### `autobright-module.zip` — Adaptive brightness

A framework RRO (`/product/overlay`) that supplies the auto-brightness curve the
GSI lacks. On Android 16 a GSI with no device overlay produces an empty
`BrightnessMappingStrategy`, so `mAutoBrightnessAvailable=false` — the
**Adaptive brightness toggle is hidden in Settings and Quick Settings** and
brightness never tracks the light sensor (the sensor itself works fine).

The module sets `config_automatic_brightness_available=true` and injects
`config_autoBrightnessLevels` / `config_autoBrightnessLcdBacklightValues`,
restoring the Settings + QS toggles **and** making brightness follow lux.

The curve is tuned for the Y700 panel. To customise, edit
`config_autoBrightnessLevels` (lux thresholds) and
`config_autoBrightnessLcdBacklightValues` (0–255, length = levels + 1) in the
overlay APK and rebuild.

> Dark-room tip: pair with TrebleApp's *lowest brightness* option, which lowers
> the framework minimum-brightness clamp so the bottom of the curve isn't floored.

### `notelephony-module.zip` — Disable telephony (Wi-Fi-only tablet)

The Y700 has no modem, but the GSI still declares the `android.hardware.telephony*`
features. That makes `com.android.phone` start and crash-loop on an
`ACCESS_NETWORK_STATE` `SecurityException`, and shows dead SIM / mobile / call
UI. TrebleApp's "remove telephony" toggle can't write the permission XML on the
read-only system, so it doesn't stick.

This module ships a permissions XML in `/product/etc/permissions` that marks all
telephony features `<unavailable-feature>`. Result: **SIM / mobile / call UI is
hidden and the `com.android.phone` crash-loop stops** (also recovers the
battery / CPU it was wasting).

---

## Credits

- [MisterZtr/LineageOS_gsi](https://github.com/MisterZtr/LineageOS_gsi) — base GSI patches & manifest
- [TrebleDroid](https://github.com/TrebleDroid) / [phhusson](https://github.com/phhusson) — Treble GSI
- [Evolution-X FaceUnlock](https://github.com/Evolution-X/packages_apps_FaceUnlock) / Paranoid Android Sense — FaceSense stack
