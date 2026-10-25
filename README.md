# Rooting the Hisense A10 (HNR667N) with Magisk

Root for the Hisense A10 e-ink phone. Tested on software **N2903.6.04.15.00 (build 82)**, Android 16,
security patch 2025-12-05, with Magisk **v30.7**. No data wipe is needed if your bootloader is already unlocked.

> Use at your own risk. Rooting can break OTA updates, banking/DRM apps, and in the worst case leave the phone
> unbootable until you restore the stock image. Read every step before you start.

## Downloads (see [Releases](../../releases))

| File | What it is | SHA-256 |
|---|---|---|
| `init_boot_b_stock_N2903.6.04.15.00_build82.img` | Stock init_boot, build 82 | `2e3f8fe9368d5f25c943653f7412d2d7b6f9cd13275bf2cf50dc25ca844f48ec` |
| `init_boot_b_magisk-v30.7_N2903.6.04.15.00_build82.img` | Same image, patched by Magisk v30.7 | `081193a96d2e7b85865c7b803eb5ccb77f90e607032d0ae0d0e3940fa822d6ec` |
| `boot_b_stock_N2903.6.04.15.00_build82.img` | Stock boot (kernel only, not modified by rooting) | `867c0fb47833e5cc5ac83805ed12fc98a2f9fe2c1a8fb3e09e8d16b2055af7c6` |

The stock hashes match the ones in Hisense's official build-82 OTA manifest, so the images are bit-for-bit original.

**Important:** this phone uses `init_boot` (GKI), so Magisk patches **init_boot**, not boot.

## What you need

- A PC with `adb` and `fastboot` (Android platform-tools)
- An **unlocked bootloader**. Unlocking wipes the phone. Settings → Developer options → OEM unlocking, then
  `fastboot flashing unlock`.
- USB debugging enabled
- The Magisk app: https://github.com/topjohnwu/Magisk/releases (v30.7 used here)

## Option 1: you are on build 82 (quick)

1. Check your build. It must say `.../82:user/release-keys`:
   ```
   adb shell getprop ro.build.fingerprint
   ```
   If it is a different build, **stop** and use Option 2. A mismatched init_boot may not boot.
2. Install the Magisk app:
   ```
   adb install Magisk-v30.7.apk
   ```
3. Reboot to the bootloader and check the active slot:
   ```
   adb reboot bootloader
   fastboot getvar current-slot
   ```
4. Flash the patched image to the active slot (`b` shown here; use `init_boot_a` if the slot is `a`):
   ```
   fastboot flash init_boot_b init_boot_b_magisk-v30.7_N2903.6.04.15.00_build82.img
   fastboot reboot
   ```
5. Open Magisk. When it says **"Requires additional setup"**, tap **OK**. The phone reboots once.
6. Test it: `adb shell su -c id`, then tap **Allow** on the phone. You should see `uid=0(root)`.

## Option 2: any build (dump your own init_boot)

The A10's bootloader does **not** support `fastboot boot`, and its fastbootd does **not** support `fastboot fetch`.
So you can't temp-boot a kernel or read partitions over fastboot. What does work is **DSU (Dynamic System Updates)**.
DSU boots a rooted generic system image **once**, from a file on `/data`, without touching your real system or
partitions. From there you can copy your own stock init_boot.

1. Download a **userdebug** Android 16 arm64 GSI that ships `su`. The TrebleDroid build
   `system-td-arm64-vanilla.img.xz` (release `ci-20250617`) from
   https://github.com/TrebleDroid/treble_experimentations/releases was used here. It has `ro.adb.secure=0`, so adb works
   without tapping "Allow" on the screen.
2. Unpack it, rename the image to `system.img`, and zip it **without compression**:
   ```
   xz -d system-td-arm64-vanilla.img.xz
   mv system-td-arm64-vanilla.img system.img
   zip -0 gsi.zip system.img
   adb push gsi.zip /sdcard/Download/gsi.zip
   ```
3. Turn on DSU and start the install (4 GB of temporary userdata; your real data is untouched):
   ```
   adb shell setprop persist.sys.fflag.override.settings_dynamic_system true
   adb shell am start-activity -n com.android.dynsystem/com.android.dynsystem.VerificationActivity \
       -a android.os.image.action.START_INSTALL -d file:///storage/emulated/0/Download/gsi.zip \
       --el KEY_USERDATA_SIZE 4294967296
   ```
   When it finishes, the DSU notification offers **Restart**. Tap it, or run:
   ```
   adb shell am start-service -n com.android.dynsystem/.DynamicSystemInstallationService \
       -a com.android.dynsystem.ACTION_REBOOT_TO_DYN_SYSTEM
   ```
4. The phone boots into the GSI. Get root and copy the partitions (`_b` = your active slot):
   ```
   adb root
   adb shell 'for p in init_boot_b boot_b vendor_boot_b vbmeta_b; do dd if=/dev/block/by-name/$p of=/data/local/tmp/$p.img bs=1M; done'
   adb pull /data/local/tmp/init_boot_b.img
   adb pull /data/local/tmp/boot_b.img
   ```
   **Keep these files safe.** `init_boot_b.img` is your way back to stock.
5. Leave the GSI. The DSU boot is one-shot, but make sure:
   ```
   adb shell gsi_tool disable
   adb reboot
   ```
   Back on your normal system, remove the leftover image with **Discard** in the DSU notification, or with
   Settings → System → Developer options → DSU. Then delete `gsi.zip`.
6. Patch it: install Magisk, copy `init_boot_b.img` to the phone, then in Magisk go to
   **Install → Select and Patch a File**. Pull the `magisk_patched-*.img` from `Download/`.
7. Flash it as in Option 1, steps 3–6.

## Undo / recovery

- **Unroot or fix a bootloop:** hold Power + Volume Down to reach the bootloader, then flash the stock image back:
  ```
  fastboot flash init_boot_b init_boot_b_stock_N2903.6.04.15.00_build82.img
  fastboot reboot
  ```
  init_boot only holds the Android ramdisk, so the bootloader is always still reachable.
- **Before installing a Hisense OTA update:** flash the stock init_boot back first, or the update will fail.
  Afterwards, re-root using Option 2 for the new build.
- **Never** run `fastboot flashing lock` with a modified image installed. Re-locking would leave the phone
  unbootable and wipe it.

## Device notes

| | |
|---|---|
| Model | Hisense A10, `HNR667N`, SoC QCM6690 |
| Fingerprint | `Hisense/HNR667N/HNR667N:16/BQ2A.260225.001-BP2A.250705.008/82:user/release-keys` |
| Vendor / boot | Android 14 (`UKQ1.260305.001`), kernel `6.1.145-android14-11` (GKI) |
| Partitions | A/B, virtual A/B, `init_boot` present (8 MiB) |
| Stock firmware | Not published. Hisense's server only serves incremental OTAs, so dumping your own is the only way to get images |
