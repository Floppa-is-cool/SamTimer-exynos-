# SamTimer Fork

![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)
![Devices](https://img.shields.io/badge/devices-Exynos%20Galaxy-1428A0?logo=samsung&logoColor=white)
![Root](https://img.shields.io/badge/root-KernelSU--Next-black)
![Status](https://img.shields.io/badge/status-experimental-orange)
![License](https://img.shields.io/badge/license-none-lightgrey)

One-tap Android root for Samsung Galaxy devices, built around the **CVE-2026-98260** timer exploit (posix-cpu-timers UAF) and a **KernelSU-Next** late-load module with Samsung KDP/RKP/DEFEX compatibility.

> Based on the **CVE-2026-98260-aurora** repo — its timer LPE port (real-cred zero-write / fake-cred ksud late-load chains) is what `app/src/main/cpp/timer.c` evolved from.

> **Status:** 🚧 TODO — fix modules

---

## Table of Contents

- [Disclaimer](#️-disclaimer)
- [Confirmed Device Support](#-confirmed-device-support)
- [Overview](#-overview)
- [Features](#-features)
- [Requirements](#-requirements)
- [Install Guide](#-install-guide)
- [Build From Source](#️-build-from-source)
- [Usage](#️-usage)
- [Temporary Root Safety Guide](#-temporary-root-safety-guide)
- [How It Works (Chain)](#️-how-it-works-chain)
- [Credits / Upstream](#-credits--upstream)
- [License](#-license)

---

## ⚠️ Disclaimer

Use at your own risk. The authors take no responsibility for data loss, bricked devices, or anything else that happens to hardware you point this at.

This is a **temporary** root method for devices you own or are explicitly authorized to test. The exploit is a probabilistic kernel UAF — a failed run can leave the system unstable until the next reboot. Rebooting always restores the phone to a clean, unrooted state.

> ⚠️ **CRITICAL REBOOT WARNING:** Do not use the normal "Restart" button or `adb reboot` after rooting. A soft reboot will cause a kernel panic due to the parked timer UAF nodes. To safely restart, **Power Off** (hold the power button), then turn the device back on manually.
fix is coming soon

---

## 🚀 Confirmed Device Support

Tested and confirmed working on:

- **Galaxy S24 FE (Exynos 2400e)** — `SM-S721B` / `r12s`, kernel `6.1.162-android14-11`, One UI 9 *(uses the e1s / Exynos 2400 profile — identical architecture)*
- **Galaxy S24 (Exynos 2400)** — `SM-S921N` / `e1s` *(original upstream test device)*
- **S24 / S24+ / S24 FE / S25 FE**  confirmed on both the September and October security patches

---

## 📋 Overview

| | |
|---|---|
| Exploit | CVE-2026-98260 (`timer.c` payload, built by Gradle) |
| Root module | KernelSU-Next 3.4.2 (`ksud-next-generic-kdp`, android14-6.1 no-patch-text) |

## ✨ Features

- Material 3 Compose UI (English / Korean, follows the system locale)
- Live root badge — detects the loaded KernelSU module every few seconds
- Confirmation dialog before every run; **re-root** button stays available when already rooted
- **Auto-root after reboot** — boot receiver → WorkManager chain that waits 3 minutes for the system to settle, waits for Shizuku, then runs the chain headless with progress notifications. A failed run disables the option itself (boot-loop protection); a manual run pressed inside the 3-minute post-boot window is held back with a countdown instead of being thrown at a still-busy system
- **Safe boot (block modules)** — one-run panic recovery for a bad module: the payload reads the toggle straight out of the app's prefs XML at stage time (DirtyFrag pattern — no relay file). Enabled: every KernelSU module dir gets a `disable` flag and `service.d` / `post-fs-data.d` are parked; turn the switch off and the next run restores everything (the auto-root chain re-establishes root every boot regardless, so the fix reaches you even if the phone boot-looped)
- The exploit payload is compiled automatically by Gradle — `assembleDebug` always ships a fresh binary

---

## 📦 Requirements

1. **Shizuku**, running via wireless debugging. For auto-root: enable Shizuku's boot auto-start and grant its permission to SamTimer once.
2. **KernelSU-Next manager** (`com.rifsxd.ksunext`) installed — `su` is only granted to apps the manager authorizes.

---

## 📥 Install Guide

### Step 1 — Download the 3 required apps

1. **SamTimer APK** — the pre-built `app-debug.apk` from this repo's [Releases page](https://github.com/Floppa-is-cool/SamTimer-exynos-/releases/tag/Realese)
2. **KernelSU-Next APK** — latest release from KernelSU-Next on GitHub *(regular KernelSU will not work)*
3. **Shizuku** — from the Google Play Store

### Step 2 — Set up Shizuku

1. Settings → About phone → Software information → tap Build number 7 times to unlock Developer Options
2. Settings → Developer Options → turn on Wireless Debugging
3. Open Shizuku and follow the on-screen steps to start it via Wireless Debugging
4. Confirm Shizuku shows **"Running"** in the notification shade

### Step 3 — Run the exploit

1. Install the KernelSU-Next app and the SamTimer app
2. Open SamTimer
3. Allow the Shizuku permission prompt
4. Tap **Root now** → confirm
5. Wait 1–7 minutes. The screen may look frozen — this is normal. Keep the app open and the screen on

### Step 4 — Verify root

1. Open KernelSU-Next
2. It should show **"Working"** with a green badge
3. You now have full temporary root

---

## 🛠️ Build From Source

```sh
./gradlew assembleDebug
# APK: app/build/outputs/apk/debug/app-debug.apk
```

- `app/src/main/cpp/timer.c` is compiled into `app/src/main/jniLibs/arm64-v8a/libtimer.so` by the `buildPayload` task (NDK is auto-detected: `ANDROID_NDK_HOME` → `local.properties` → SDK → `~/android-ndk-r27c`)
- `libksud.so` is a prebuilt KernelSU-Next late-load daemon
- `./gradlew clean` removes all generated outputs (the prebuilt `libksud.so` is kept)

## ▶️ Usage

1. Start Shizuku, open SamTimer
2. Tap **Root now** → confirm → wait 1–7 minutes
3. Grant `su` to your apps in the KernelSU-Next manager
4. Optionally enable **Auto-root after reboot**

---

## 🔐 Temporary Root Safety Guide

> **Before installing any module you're unsure about:** remove your screen lock and confirm Download Mode works first. If you can't verify that, don't install it. A UI tweak isn't worth bricking your phone.

### 🚨 Do Not Flash (High Brick Risk)

These modify persistent partitions and can trigger the D2 hardware lock or cause a permanent brick:

| Risk | Description |
|---|---|
| **Metamodules** (NoMount, Hybrid Mount, meta-overlayfs, etc.) | KernelSU uses these to mount system files, but on temporary root they're dangerous — they run with root access and can mount anything; a misconfigured one can corrupt the system partition and cause a bootloop or hard brick |
| **Partition writers** | Anything writing to `/system`, `/vendor`, `/boot`, or `/product` |
| **Boot script replacers** | Modules that modify `init.rc` or replace boot scripts |
| **Pre-boot starters** | Modules that need to launch before Android boots |
| **Magisk boot patches** | Patching the boot image writes directly to the boot partition — avoid |

### ✅ Safe Modules (Runtime Only)

These run in memory, touch no partitions, and clear on reboot:

- **LSPosed & Zygisk modules** — runtime app hooks
- **System UI tweaks** — cosmetic changes like Dynamic Island
- **Script-only modules** — anything limited to scripts or `system.prop`

### 🚑 If Something Goes Wrong

- **Stuck on "Reboot Device – D2" with no access to Download Mode:** a hardware-level lock has been tripped — you'll need a new motherboard to recover
- **If you can still access Download Mode:** flash the stock firmware image

### 📎 Modules Guide Reference

Community thread on safe vs. risky modules for temp root: [r/androidroot — Modules Guide](https://www.reddit.com/r/androidroot/s/5ueXyhcuyz)

*Stay safe — a bricked phone isn't worth a UI tweak.*

---

## ⚙️ How It Works (Chain)

1. `timer.c` leaks the child cred through the kmem trace, clears the DEFEX `creds_hash` slot, installs a forged uid-0 cred (the `selinux_state` write lands at -7 so `initialized` survives — an 8-byte zero at offset 0 kills the policy-loaded flag and breaks every property write)
2. The root worker adopts `u:r:init:s0`, binds `ksud` over `/system/bin/logcat` inside a private mount namespace (KDP-safe exec) and runs `late-load --package-name com.rifsxd.ksunext`
3. KernelSU-Next loads (no-patch-text build — live text patching panics on Exynos 2400); SELinux is restored to enforcing by `ksud`
4. Success is verified through the KSU driver's `reboot()` backdoor ioctl
5. Failed-plant zombies are made reap-safe first — their posix-cpu-timer queue leftmost pointers are zeroed through the write primitive, so `posix_cpu_timers_exit` finds empty queues and the reap cannot touch the stale node — then reaped on the spot. Nothing is left parked, and there is never a zombie for init to reap later; a park is only a last-resort fallback when a repair plant cannot land

---

## 🙏 Credits / Upstream

- **CVE-2026-98260-aurora** — this app is built on the aurora repo's timer LPE port: the `timer.c` payload started there and lives in this project as `app/src/main/cpp/timer.c`
- **yoro1836/SamTimer** — original upstream repository
- **BuSung-dev/Root-My-Galaxy** — the verified Samsung KSU late-load flow this app mirrors (`su_daemon.c` shape, `.ksud-stage` staging, manager pairing)
- **BuSung-dev/Root-My-Galaxy-Payloads** — per-target `ksud`/`kernelsu.ko` pairs and the Samsung KDP/RKP/DEFEX port methodology (exact-release vermagic, no-patch-text for Exynos)
- **rushiranpise/Root-My-Galaxy-Payloads** — Samsung delta patches for KernelSU 3.2.5–3.4.2, KernelSU-Next and ReSukiSU; `ksud-next-generic-kdp` (KernelSU-Next 3.4.2) used here
- **tiann/KernelSU · KernelSU-Next/KernelSU-Next** — the root module lineage
- **DirtyFrag** (CVE-2026-43284, DFRoot lineage) — the autorun / failed-run-interlock pattern
- **Shizuku** — shell-uid execution without a PC

---

## 📄 License

This is a fork of [yoro1836/SamTimer](https://github.com/yoro1836/SamTimer), which does not ship a `LICENSE` file. No license is granted here either — all rights remain with the original author(s) and contributors. Reach out to them before reusing or redistributing this code beyond personal/testing use.
