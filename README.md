# Hisense A10 (HNR667N) root with Magisk

How I rooted my Hisense A10 e-ink phone, plus the stock and patched images. Done on firmware
**N2903.6.04.15.00 (build 82)**, Android 16, December 2025 patch, with Magisk 30.7.

There's no stock firmware for this phone anywhere online. Hisense only pushes incremental OTAs, so I dumped the
images myself and checked them against the hashes in Hisense's own OTA manifest. They're bit-for-bit original.

The usual disclaimer: this is your phone and your risk. Unlocking wipes everything, and banking/tap-to-pay apps
may stop working.

## Files

Grab them from [Releases](../../releases):

- `init_boot_b_stock_N2903.6.04.15.00_build82.img`: stock init_boot (keep this, it's your way back)
- `init_boot_b_magisk-v30.7_N2903.6.04.15.00_build82.img`: the same image patched with Magisk 30.7
- `boot_b_stock_N2903.6.04.15.00_build82.img`: stock boot (kernel). You don't need to flash it, it's just a backup

SHA-256 sums are in `SHA256SUMS`.

One thing that trips people up: the A10 is a GKI device with an `init_boot` partition. Magisk patches
**init_boot**, not boot.

You'll need adb/fastboot (Google's platform-tools) and the Magisk APK from
[the official GitHub](https://github.com/topjohnwu/Magisk/releases).

## 1. Unlock the bootloader

**This wipes the phone.** Back up first.

1. Settings → About phone, tap the build number 7 times to get Developer options.
2. In Developer options, turn on **OEM unlocking** and **USB debugging**.
3. Plug into the PC and reboot to the bootloader:
   ```
   adb reboot bootloader
   ```
4. Unlock:
   ```
   fastboot flashing unlock
   ```
   Confirm on the phone (volume keys to select, power to confirm). It wipes and reboots.
5. Go through setup again, then turn Developer options and USB debugging back on.

From now on you'll get the orange "device is unlocked" warning at every boot. That's normal.

## 2a. Root it (if you're on build 82)

Check your build first:
```
adb shell getprop ro.build.fingerprint
```
It needs to end in `/82:user/release-keys`. If it doesn't, use 2b instead. Flashing an init_boot from a different
build is asking for trouble.

```
adb install Magisk-v30.7.apk
adb reboot bootloader
fastboot getvar current-slot
fastboot flash init_boot_b init_boot_b_magisk-v30.7_N2903.6.04.15.00_build82.img
fastboot reboot
```
(If `current-slot` says `a`, flash `init_boot_a` instead.)

Open Magisk. It'll say it needs additional setup. Tap OK and it reboots once. That's it. Check with
`adb shell su -c id` and allow the prompt on the phone.

## 2b. Root it (any build): dump your own init_boot

This is what I actually had to do. The A10's bootloader doesn't support `fastboot boot`, and fastbootd doesn't
support `fastboot fetch`, so the usual ways of getting an image off the phone don't work. What does work is
DSU (Dynamic System Updates). It lets you boot a generic Android image once from a file on `/data`, without
touching your real system. If that image is a userdebug build, you get root and can just `dd` the partitions.

I used TrebleDroid's `system-td-arm64-vanilla.img.xz` (release `ci-20250617`) from
https://github.com/TrebleDroid/treble_experimentations/releases. It's Android 16 userdebug, and adb works
without having to tap "allow" on the screen.

```
xz -d system-td-arm64-vanilla.img.xz
mv system-td-arm64-vanilla.img system.img
zip -0 gsi.zip system.img            # -0 = no compression, it's faster
adb push gsi.zip /sdcard/Download/gsi.zip

adb shell setprop persist.sys.fflag.override.settings_dynamic_system true
adb shell am start-activity -n com.android.dynsystem/com.android.dynsystem.VerificationActivity \
    -a android.os.image.action.START_INSTALL -d file:///storage/emulated/0/Download/gsi.zip \
    --el KEY_USERDATA_SIZE 4294967296
```

When it's done, hit Restart in the DSU notification, or:
```
adb shell am start-service -n com.android.dynsystem/.DynamicSystemInstallationService \
    -a com.android.dynsystem.ACTION_REBOOT_TO_DYN_SYSTEM
```

Once it's booted into the GSI:
```
adb root
adb shell 'for p in init_boot_b boot_b vendor_boot_b vbmeta_b; do dd if=/dev/block/by-name/$p of=/data/local/tmp/$p.img bs=1M; done'
adb pull /data/local/tmp/init_boot_b.img
adb pull /data/local/tmp/boot_b.img
adb shell gsi_tool disable
adb reboot
```
(Swap `_b` for `_a` if that's your active slot.)

You're back on your normal system now. Get rid of the DSU image with Discard in its notification, and delete
`gsi.zip`.

Then patch it the normal way: copy `init_boot_b.img` to the phone, then in Magisk go to Install → Select and Patch
a File. Pull the `magisk_patched-*.img` out of Downloads and flash it like in 2a.

## Going back / if it bootloops

Hold power + volume down to get into the bootloader, then:
```
fastboot flash init_boot_b init_boot_b_stock_N2903.6.04.15.00_build82.img
fastboot reboot
```
init_boot only holds the Android ramdisk, so the bootloader always stays reachable.

Flash stock init_boot back **before** taking a Hisense OTA too, otherwise the update fails.

Don't relock the bootloader with a patched image installed. That's how you brick it.

## Phone details

- Model HNR667N, Qualcomm QCM6690
- `Hisense/HNR667N/HNR667N:16/BQ2A.260225.001-BP2A.250705.008/82:user/release-keys`
- Vendor/boot is Android 14 (`UKQ1.260305.001`), kernel 6.1.145 GKI
- A/B with virtual A/B; init_boot is 8 MB
