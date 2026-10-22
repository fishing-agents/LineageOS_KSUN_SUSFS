# Play Integrity debugging: OnePlus 11 (salami), LineageOS 24, KernelSU Next + SUSFS

Run these from a desktop with the phone connected over USB (`adb devices` shows it, and adb root is granted in the KernelSU Next manager for the Shell app).

## Stack

- **Kernel:** LineageOS 24.0 sm8550 kernel + KernelSU Next 33388 + SUSFS v2.3.0, flashed to `boot`. `init_boot` is stock.
- **Zygisk:** Zygisk Next (`zygisksu`)
- **PlayIntegrityFix:** KOWX712 fork, inject-s (config in `pif.prop`)
- **Keystore spoofing:** TEESimulator (`teesim`) or none, using PIF's `spoofProvider`
- **Tricky Addon:** optional. Released versions don't support TEESimulator (see "Known issues").

## What we found so far (2026-10-22)

1. **The modules load.** Zygisk Next runs, and PIF injects into `com.google.android.gms.unstable`. The log shows both the Java and native spoofing (`PIF/Java:DG`, `PIF/Native`).
2. **The fingerprint is a Canary build.** PIF's fetch takes the newest build listed on developer.android.com, which right now is Canary:
   ```
   google/<device>_beta/<device>:CANARY/ZP11.260918.007/16484274:user/release-keys
   ```
   PIF then reports `RELEASE` as `CANARY` instead of a number (log: `[RELEASE]: 17 -> CANARY`). That's the main suspect for Basic failing. Changing the device (Pixel 6a, 7 or 8) doesn't help, because they all get the same Canary build.
3. **`/data/adb/pif.prop` takes precedence over the module's own `pif.prop`.** Edit that one.
4. **Not checked yet:** the TEESimulator Root of Trust (`deviceLocked`, `verifiedBootState`). TEESimulator reads the real verified-boot state from the phone rather than faking it, so with an unlocked bootloader it may sign "unlocked / unverified". That blocks Device and may affect Basic.
5. **Ruled out:** partition layout. If it were wrong, the phone wouldn't boot or the modules wouldn't load.

## Order to check things in

Go top to bottom and stop at the first step that fails.

### 1. Phone and kernel

```sh
adb shell getprop ro.lineage.version
adb shell uname -r          # expect 5.15.220-ga681efd8f0bf for the 2026-10-16 build
adb shell su -c 'ls -a /data/adb/modules/'
```

- `uname -r` must match the release in the kernel build's `info.txt`. If it doesn't, the kernel was built for a different LineageOS build. Rebuild it or reflash stock `boot`.
- Expected modules: `zygisksu`, `playintegrityfix`, and `teesim` if you use it. Only one Zygisk implementation. Never both `tricky_store` and `teesim` enabled.

### 2. Zygisk Next

```sh
adb shell su -c 'grep description /data/adb/modules/zygisksu/module.prop'
adb shell su -c 'ps -A | grep -i zygisk'
```

- The description line should show zygote64 injected and the daemon running.
- Zygisk Next denylist policy should be **disabled**. Google Play Services and the Play Store should have no unmount profile in the KernelSU Next manager.

### 3. PIF config

```sh
adb shell su -c 'cat /data/adb/pif.prop 2>/dev/null || cat /data/adb/modules/playintegrityfix/pif.prop'
adb shell unzip -l /system/etc/security/otacerts.zip | grep -oE 'testkey|releasekey'
```

Expected values:

- `FINGERPRINT`: Android version is a number (`:16/` or `:17/`), **not `:CANARY/`**
- `SECURITY_PATCH`: from the same build as the fingerprint
- `spoofBuild=true`
- `spoofProps=true`
- `spoofProvider`: `false` if TEESimulator/TrickyStore is active, `true` if neither is
- `spoofSignature`: `false`, unless otacerts shows `testkey`
- `spoofVendingSdk=false`

### 4. Is PIF injecting into Play Services?

```sh
adb shell su -c 'sed -i "s/^DEBUG=false/DEBUG=true/" /data/adb/pif.prop'
adb shell su -c 'killall com.google.android.gms.unstable com.android.vending; logcat -c'
# run the integrity checker on the phone now
adb logcat -d | grep -iE 'PIF' | tail -60
```

How to read it:

- **No PIF lines:** PIF isn't injected. Check Zygisk (step 2) and the denylist/unmount settings.
- **`Set 'RELEASE' to 'CANARY'` or `[RELEASE]: 17 -> CANARY`:** Canary fingerprint. Fix step 3.
- **`Dex payload skipped because spoofProvider and spoofSignature are false`:** expected with TEESimulator. Without TEESimulator it means `spoofProvider` is still off.

`killall: ... No such process` is harmless. It means the process wasn't running.

Turn debug back off when you're done:

```sh
adb shell su -c 'sed -i "s/^DEBUG=true/DEBUG=false/" /data/adb/pif.prop'
```

### 5. TEESimulator

```sh
adb shell su -c 'ls -la /data/adb/teesim/; cat /data/adb/teesim/config.json'
adb shell su -c 'pidof teesim && echo daemon-ok; grep ^version /data/adb/modules/teesim/module.prop'
adb shell su -c 'ls /data/adb/modules/teesim/'   # there should be no disable or remove file
```

- `keybox.xml` must exist and contain both an RSA and an EC key, each with a chain of at least 2 certificates.
- The `config.json` profile `apps` must include `com.google.android.gms` and `com.android.vending`, with `mode: patch`.
- Use stable **v4.0**. Recent canaries have open bugs (#307, #310, #311).

On the phone, open the **Key Attestation** app (`io.github.vvb2060.keyattestation`, targeted by default). It should show:

- the keybox chain (not the phone's own Qualcomm chain), not revoked
- **Root of Trust:** `deviceLocked` and `verifiedBootState`. Write these down. `false` / `Unverified` means TEESimulator is passing the real unlocked state through, so Device won't pass.

### 6. Isolate the keystore side

Turn TEESimulator off and let PIF fake the keystore:

```sh
adb shell su -c 'touch /data/adb/modules/teesim/disable; sed -i "s/^spoofProvider=false/spoofProvider=true/" /data/adb/pif.prop'
adb reboot
# after boot: step 4 again, then run the checker
```

Undo:

```sh
adb shell su -c 'rm -f /data/adb/modules/teesim/disable; sed -i "s/^spoofProvider=true/spoofProvider=false/" /data/adb/pif.prop'
adb reboot
```

Do this test **with a non-Canary fingerprint**. Otherwise a failure doesn't tell you anything.

- **Basic passes with spoofProvider but not with TEESimulator:** the TEESimulator keybox or Root of Trust is the problem. Try TrickyStoreOSS, which rewrites the Root of Trust to locked/verified.
- **Basic fails both ways:** fingerprint or injection problem (steps 3 and 4).

## Expected results

- **No modules:** fails everything (normal for an unlocked custom ROM)
- **PIF with a good fingerprint and spoofProvider:** Basic, sometimes Device
- **PIF plus TEESimulator/TrickyStore with a valid keybox:** Device, rarely Strong
- **Canary fingerprint:** probably fails even Basic

Checker notes: use a real Play Integrity API checker. Force-stop Google Play Services and the Play Store between tries, and wait about a minute. Google rate-limits these checks.

## Known issues

- **MMRL crashes on Android 17:** `No interface method getInstalledPackages`. Fixed only in MMRL CI builds ([#652](https://github.com/MMRLApp/MMRL/issues/652), [#658](https://github.com/MMRLApp/MMRL/issues/658)). Use the KernelSU Next manager's Modules tab instead.
- **Tricky Addon removes itself:** releases v4.4 and v5.0-beta.4 only detect `tricky_store` or `oh_my_keymint`. With TEESimulator only, `post-fs-data.sh` writes `TA_utl/remove` on every boot. TEESimulator support (#154) is only in CI builds. Either use TEESimulator's own WebUI, or use a Tricky Addon CI build from main.
- **ReZygisk / BreZygisk:** no support for the Android 17 `zygote_next` change yet (ReZygisk #374). Use Zygisk Next.
- **TrickyStore and TEESimulator** must not both be enabled. Both hook keystore2. TEESimulator's installer disables TrickyStore automatically.

## Collect everything in one go

```sh
#!/bin/sh
# pi-dump.sh — run on the desktop with the phone connected; writes pi-dump-<date>.txt
out="pi-dump-$(date +%Y%m%d-%H%M%S).txt"
s() { echo "### $*" >>"$out"; adb shell su -c "$*" >>"$out" 2>&1; echo >>"$out"; }
s 'getprop ro.lineage.version; uname -r; getprop ro.boot.verifiedbootstate; getprop ro.boot.flash.locked'
s 'ls -la /data/adb/modules/ /data/adb/modules/*/disable /data/adb/modules/*/remove'
s 'grep -H -E "^(version|description)=" /data/adb/modules/*/module.prop'
s 'ps -A | grep -iE "zygisk|teesim"'
s 'cat /data/adb/pif.prop; echo ---; cat /data/adb/modules/playintegrityfix/pif.prop'
s 'ls -la /data/adb/teesim/; cat /data/adb/teesim/config.json'
s 'unzip -l /system/etc/security/otacerts.zip'
adb shell su -c 'sed -i "s/^DEBUG=false/DEBUG=true/" /data/adb/pif.prop; killall com.google.android.gms.unstable com.android.vending; logcat -c' 2>/dev/null
echo ">>> Run the Play Integrity check on the phone, then press Enter"; read _
echo "### logcat" >>"$out"
adb logcat -d | grep -iE 'PIF|PlayIntegrity|teesim|keystore2|zygisk' | tail -200 >>"$out"
adb shell su -c 'sed -i "s/^DEBUG=true/DEBUG=false/" /data/adb/pif.prop'
echo "wrote $out"
```

The dump doesn't include the keybox contents. Don't share `keybox.xml` or `admin.token`.
