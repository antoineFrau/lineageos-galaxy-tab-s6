# LineageOS 23.2 on the Samsung Galaxy Tab S6 Lite (SM-P610)

A complete, tested walkthrough for replacing One UI with **LineageOS 23.2
(Android 16)** on the Wi-Fi Galaxy Tab S6 Lite 2020 — codename `gta4xlwifi` —
from a Linux host, with optional Google Play Services contained in a work
profile.

Written from an actual install performed on 2026-10-19/20 against
`lineage-23.2-20261017`. Every command here was run on the real device; the
pitfalls section documents failures that actually happened, not hypotheticals.

---

## Table of contents

- [Scope and compatibility](#scope-and-compatibility)
- [What this costs you](#what-this-costs-you)
- [Prerequisites](#prerequisites)
- [Step 1 — Verify the device](#step-1--verify-the-device)
- [Step 2 — Unlock the bootloader](#step-2--unlock-the-bootloader)
- [Step 3 — Download and verify the images](#step-3--download-and-verify-the-images)
- [Step 4 — Enter download mode](#step-4--enter-download-mode)
- [Step 5 — Flash LineageOS recovery](#step-5--flash-lineageos-recovery)
- [Step 6 — Boot into recovery immediately](#step-6--boot-into-recovery-immediately)
- [Step 7 — Wipe and sideload](#step-7--wipe-and-sideload)
- [Step 8 — Google Play Services (optional)](#step-8--google-play-services-optional)
- [Step 9 — First boot](#step-9--first-boot)
- [Step 10 — A real Linux userland](#step-10--a-real-linux-userland)
- [Pitfalls and troubleshooting](#pitfalls-and-troubleshooting)
- [Sources](#sources)

---

## Scope and compatibility

This guide is for **exactly one model**:

| | |
|---|---|
| Marketing name | Galaxy Tab S6 Lite (2020), Wi-Fi |
| Model number | `SM-P610` |
| Codename | `gta4xlwifi` |
| SoC | Exynos 9611 (`LSI9610` in the bootloader) |
| Target OS | LineageOS 23.2 (Android 16) |

> [!CAUTION]
> The "Tab S6 Lite" name covers three unrelated devices. **`SM-P615`** is the
> LTE 2020 model (`gta4xl`), **`SM-P613`/`SM-P619`** is the 2022 Snapdragon
> 720G refresh, and **`SM-P620`/`SM-P625`** is the 2024 model. The LineageOS
> wiki requires an *exact* model match. Flashing the wrong build can hard-brick
> the tablet. Check Settings → About tablet before continuing.

### Why not postmarketOS?

If you came here wanting "real" Linux rather than Android, the honest answer
for this tablet is that postmarketOS is not yet a practical option. The
device page exists, but the device package was never merged into `pmaports`,
so it cannot be selected in `pmbootstrap` and no prebuilt images are offered.
Audio, GPU acceleration, the camera and the SD card are all listed as broken,
on a downstream 4.14 kernel.

LineageOS plus a Debian chroot ([Step 10](#step-10--a-real-linux-userland))
gives you a working `apt` and a desktop with every piece of hardware intact.
That is the trade this guide makes.

---

## What this costs you

> [!WARNING]
> Read this section before touching anything.

- **Knox is permanently tripped.** Unlocking the bootloader blows a hardware
  e-fuse. Samsung Pay, Secure Folder and Samsung Health stop working *forever*,
  even if you reflash stock firmware and re-lock. This cannot be undone.
- **All data is erased.** Twice, in fact — once by the unlock, once by the
  format in Step 7.
- **Warranty.** Draw your own conclusions.
- **Remove all Google accounts first.** Otherwise Factory Reset Protection can
  lock you out of your own tablet after the wipe.

---

## Prerequisites

### Required firmware version

> [!IMPORTANT]
> LineageOS 23.2 for this device requires the tablet to be on **stock Android
> 13** before you start, even though you are installing Android 16. This is not
> an error. The LineageOS build relies on firmware blobs shipped by that stock
> release. Installing from the wrong base can cause crashes or **permanent
> damage**.

`SM-P610`'s final official Samsung release was Android 13 (One UI 5.1.x), so a
fully updated stock tablet satisfies this. Verify it; do not assume it.

### Host packages (Arch Linux)

```bash
sudo pacman -S --needed heimdall android-tools
```

On Debian/Ubuntu: `sudo apt install heimdall-flash android-tools-adb`.

If you would rather not install anything system-wide, both tools run fine from
their upstream archives in your home directory — only the udev rule needs root.

### udev rule

Samsung download mode needs non-root USB access:

```bash
sudo tee /etc/udev/rules.d/51-samsung-download.rules >/dev/null <<'EOF'
# Samsung Galaxy devices - download mode / MTP / adb access for local user
SUBSYSTEM=="usb", ATTR{idVendor}=="04e8", MODE="0660", TAG+="uaccess"
EOF
sudo udevadm control --reload-rules
```

### A good USB connection

> [!TIP]
> Plug the tablet **directly into the computer**. Not a dock, not a hub. A
> flash that dies mid-write to the recovery partition is how tablets end up
> needing a stock-firmware rescue. Use the cable that came with the device if
> you still have it.

---

## Step 1 — Verify the device

Enable **Developer options** (Settings → About tablet → tap *Build number*
seven times), then **USB debugging**, and confirm what you are dealing with:

```bash
adb shell getprop | grep -E 'ro.product.model|ro.product.device|ro.build.version.release|ro.boot.flash.locked'
```

Expected:

```
[ro.boot.flash.locked]: [0]          # 0 = unlocked (will be [1] before Step 2)
[ro.build.version.release]: [13]     # must be 13
[ro.product.device]: [gta4xlwifi]
[ro.product.model]: [SM-P610]
```

Check that no Google accounts remain:

```bash
adb shell dumpsys account | grep -cE 'Account \{name='   # must print 0
```

---

## Step 2 — Unlock the bootloader

Skip if `ro.boot.flash.locked` is already `0`.

1. Connect to Wi-Fi (the OEM-unlock toggle needs a network check).
2. Developer options → enable **OEM unlocking**. If it is greyed out, the
   7-day Samsung-account lock is active; wait it out.
3. Power off. Hold **Volume Down + Volume Up**, then plug in the USB cable.
4. Press the button the screen maps to **Device unlock mode** / **Unlock
   Bootloader**. Confirm. The tablet wipes and reboots.
5. Run through Android setup, skipping everything, and **connect to Wi-Fi**.
6. Re-enable Developer options and **verify OEM unlocking is still on**.

> [!IMPORTANT]
> Step 5 and 6 are not optional. Samsung's VaultKeeper re-locks the bootloader
> if the device never boots and reaches a network after the unlock.

---

## Step 3 — Download and verify the images

```bash
mkdir -p ~/lineage-gta4xlwifi && cd ~/lineage-gta4xlwifi

BASE=https://mirrorbits.lineageos.org/full/gta4xlwifi/20261017
curl -LO $BASE/recovery.img
curl -LO $BASE/lineage-23.2-20261017-nightly-gta4xlwifi-signed.zip
```

**Always verify against the official API**, not against the mirror you
downloaded from:

```bash
curl -s https://download.lineageos.org/api/v2/devices/gta4xlwifi/builds \
  | python3 -c "import json,sys; [print(f['sha256'], f['filename']) for f in json.load(sys.stdin)[0]['files']]"

sha256sum recovery.img lineage-23.2-*.zip
```

For the build used here:

| File | SHA-256 |
|---|---|
| `recovery.img` | `78b287a75f4b4b18f74270bb2e7822487aab0368d51b51cdf401491d20d5dfc7` |
| `lineage-23.2-20261017-…zip` | `5c8b4eeceb207ccb39ecd6cf094e67a41f9b8be9c7bec523c06cf0f81cd11bc3` |

Newer nightlies will have different hashes — take them from the API, which is
the authoritative source.

---

## Step 4 — Enter download mode

1. Power the tablet **fully off**.
2. Hold **Volume Down + Volume Up** together.
3. **While holding**, plug in the USB-C cable.
4. At the warning screen, press the indicated button for **Continue**.

You should see `Downloading... Do not turn off target!!`.

Confirm from the host — the USB ID must be `04e8:685d`:

```bash
lsusb | grep 04e8
heimdall detect
```

`heimdall detect` also prints the model, bootloader version and temperature,
which is a free sanity check that you are talking to the right tablet.

> [!IMPORTANT]
> **Download mode allows exactly one protocol session per entry.** After any
> command that opens a session, every subsequent command fails with
> `Unexpected handshake response!` until you power-cycle back into download
> mode. `detect` is the only exception. Budget a re-entry for every single
> command below.

---

## Step 5 — Flash LineageOS recovery

```bash
heimdall flash --RECOVERY recovery.img --no-reboot
```

Success looks like this — an explicit upload with a percentage counter:

```
Uploading RECOVERY
0%1%2%4%...98%100%
RECOVERY upload successful
```

> [!CAUTION]
> **Do not use `samloader-rs` for this**, despite the LineageOS wiki currently
> recommending it. Version 2.2.0 silently fails: it initialises the protocol,
> opens the session, downloads the PIT, sends `Session(TotalBytes)` with the
> correct byte count, then closes the connection **without transferring
> anything — and exits 0**. Both `--partition RECOVERY recovery.img` and the
> auto-matching `-f recovery.img` fail identically. Heimdall v2.2.2 worked
> first try. See [Pitfalls](#pitfalls-and-troubleshooting).

The partition target is correct by construction: the device's PIT lists entry
13 as `RECOVERY` / `recovery.img` at 17360 blocks, and 17360 × 4096 =
71,106,560 bytes, exactly the size of the recovery image.

---

## Step 6 — Boot into recovery immediately

> [!WARNING]
> This step is time-sensitive. If the tablet boots into Android even once,
> stock silently restores its own recovery and you repeat Step 5.

The screen will still read `Downloading... Do not turn off target!!` after a
successful flash. That is expected, not a hang.

1. Hold **Volume Down + Power** for 8–10 seconds until the screen goes black.
2. Release **immediately** when it does.
3. Straight away, hold **Volume Up + Power**.

You are looking for the **LineageOS logo**. Any other recovery means it booted
wrong — redo Step 4 and 5.

---

## Step 7 — Wipe and sideload

In LineageOS recovery:

1. **Factory Reset → Format data / factory reset**. This removes encryption and
   erases `/data`.
2. Back to the main menu → **Apply update → Apply from ADB**.

Then from the host:

```bash
adb devices          # must show: <serial>  sideload
adb -d sideload lineage-23.2-20261017-nightly-gta4xlwifi-signed.zip
```

> [!NOTE]
> Before you select *Apply from ADB*, `adb devices` may report the tablet as
> `unauthorized`. This is harmless — formatting data erased the stored adb key
> and recovery has no dialog to approve a new one. Sideload mode does not use
> adb authentication, so it resolves itself the moment sideload starts.

**The progress display stops around 47%.** This is normal and documented. The
real success marker is the final line:

```
Total xfer: 1.00x
```

`adb: failed to read command: Success` (or `No error`, or `Undefined error: 0`)
is also a normal ending.

> [!IMPORTANT]
> **Do not reboot yet** if you want Google apps. Step 8 must happen before the
> first boot.

---

## Step 8 — Google Play Services (optional)

> [!CAUTION]
> **MindTheGapps is not sandboxed.** It installs Google Play Services as a
> *privileged system app* with deep system access — the opposite of sandboxing.
> Genuinely sandboxed Play Services, where Google's code runs as an ordinary
> unprivileged app in the normal app sandbox, is a **GrapheneOS** feature and
> exists only there, on Pixel hardware. No LineageOS build offers an
> equivalent. Anyone telling you otherwise is confusing the two projects.

You have three honest options on this tablet.

### Option A — No Google services (recommended)

Skip this step entirely. Reboot and use [F-Droid](https://f-droid.org) and
[Aurora Store](https://auroraoss.com) (an anonymous Play Store client) instead.
Best battery life, best privacy, nothing phoning home. Some banking apps and
DRM video will refuse to run.

### Option B — MindTheGapps, then contain it in a work profile

This is the closest practical equivalent to "sandboxed Play Services" on
LineageOS. Play Services still runs privileged, but a work profile puts it in a
separate Android user with its own encrypted storage, so it cannot read your
personal profile's contacts, files or app data, and you can freeze the entire
profile with one toggle.

It is **isolation, not sandboxing**. Worth being precise about the difference.

**Install the package before first boot:**

```bash
# LineageOS 23 (Android 16) → MindTheGapps 16.0.0, arm64
curl -LO https://github.com/MindTheGapps/16.0.0-arm64/releases/latest/download/MindTheGapps-16.0.0-arm64-20260915_222150.zip
curl -LO https://github.com/MindTheGapps/16.0.0-arm64/releases/latest/download/MindTheGapps-16.0.0-arm64-20260915_222150.zip.sha256sum
sha256sum -c MindTheGapps-16.0.0-arm64-*.zip.sha256sum
```

Check the [releases page](https://github.com/MindTheGapps/16.0.0-arm64/releases/latest)
for the current filename first — it is date-stamped and changes.

In recovery, **Apply update → Apply from ADB** again, then:

```bash
adb -d sideload MindTheGapps-16.0.0-arm64-*.zip
```

> [!NOTE]
> `Signature verification failed` is **expected** here — add-on packages are
> not signed with LineageOS's key. Select **Yes**. (By contrast, that same
> error on the ROM zip in Step 7 means a corrupt download.)

**Then, after first boot**, contain it:

1. Install [Shelter](https://f-droid.org/packages/net.typeblog.shelter/) from F-Droid.
2. Open Shelter and let it create the work profile.
3. Move or install Google apps **into the work profile only**.
4. Use the Shelter quick-settings tile to freeze the profile when not needed.

Caveat worth knowing up front: the personal profile cannot see contacts or
files belonging to the work profile. That separation is the entire point, but
it does mean Google Contacts in the work profile will not populate your
personal dialer. Shelter can clone individual apps across the boundary when you
need it.

### Option C — microG

A free-software reimplementation of the Play Services APIs. It can impersonate
Google's signature to satisfy apps, without running Google's code. On stock
LineageOS, microG works best installed as a **user app**; full functionality
needs signature spoofing, which stock LineageOS does not ship — that requires a
fork such as LineageOS for microG or DivestOS.

Lower compatibility than real Play Services, particularly for banking apps and
anything using Play Integrity. Push notifications work but need setup.

### Choosing

| | No GApps | MindTheGapps + Shelter | microG |
|---|---|---|---|
| Play Store | ✗ (use Aurora) | ✓ | partial |
| Banking apps | mostly ✗ | ✓ | often ✗ |
| Google code runs privileged | ✗ | ✓ | ✗ |
| Isolated from personal data | n/a | ✓ (work profile) | n/a |
| Battery impact | none | noticeable | small |
| Reversible without a wipe | — | ✗ | ✓ |

> [!IMPORTANT]
> Option B is a one-way door. Adding or removing MindTheGapps later means
> repeating the wipe in Step 7. Decide now.

---

## Step 9 — First boot

Back arrow → **Reboot system now**.

First boot takes up to 15 minutes. A long boot animation is normal — do not
interrupt it. Much longer than that suggests a missed step.

Updates arrive over the air afterwards via **Settings → System → Updater**;
no further flashing needed.

The S Pen works out of the box — it is handled by the kernel driver, which is
fully supported on this device.

---

## Step 10 — A real Linux userland

A chroot gives you `apt`, a shell and desktop apps with no risk and no root.
You lose kernel-level access; you keep working audio, GPU and cameras.

```bash
# In Termux — install it from F-Droid, NOT the Play Store build, which is abandoned
pkg update && pkg install proot-distro x11-repo
proot-distro install debian
proot-distro login debian
```

For a graphical desktop, add [Termux:X11](https://github.com/termux/termux-x11)
and inside the Debian container:

```bash
apt update && apt install -y xfce4 xfce4-terminal dbus-x11
```

Alternatives: **Andronix** or **Linux Deploy** wrap the same mechanism in a GUI.

---

## Pitfalls and troubleshooting

### `samloader-rs` exits 0 but nothing is flashed

The failure that cost the most time during this install. Symptom — note the
exit code:

```
Sending command: Session(TotalBytes { total_bytes: 0x43D0000 })
Received response: Response { response_type: 0x64, value: 0x00 }

Ending session...
```

The byte count is correct, the partition exists, the file resolves — and not a
single transfer command is emitted. **Exit code 0.**

**Rule: if you do not see a percentage counter, the flash did not happen**,
regardless of what the exit code says. Use Heimdall.

### `ERROR: Unexpected handshake response!`

A session was already used since the last download-mode entry. Power-cycle back
into download mode (Step 4). One session per entry, always.

### `samloader` argument forms

If you use it for read-only inspection, the flags are not positional:

```bash
samloader dump-pit --output device.pit     # not: dump-pit device.pit
samloader print-pit -f device.pit          # not: --input
```

Dump the PIT once, then parse it offline — this costs no extra sessions:

```bash
samloader print-pit -f device.pit | grep 'Partition Name:'
```

### Recovery was overwritten by stock

You let the tablet boot into Android after Step 5. Repeat Steps 4–6 and be
faster with the key combo.

### Sideload stalls at 47%

Not a stall. See Step 7 — `Total xfer: 1.00x` is the real result.

### Signature verification failed

On the **ROM** zip: corrupt or unofficial download. Re-verify the SHA-256
against the LineageOS API and download again.
On an **add-on** zip (GApps): expected. Select Yes.

### Flash dies partway through

Almost always USB. Try a different cable, a different physical port, and get
rid of any dock or hub. The tablet is recoverable from download mode as long as
the bootloader is intact — reflash stock firmware with Heimdall or Odin if
needed.

---

## Sources

- [LineageOS install guide — gta4xlwifi](https://wiki.lineageos.org/devices/gta4xlwifi/install/)
- [LineageOS downloads — gta4xlwifi](https://download.lineageos.org/devices/gta4xlwifi/)
- [LineageOS Google apps page](https://wiki.lineageos.org/gapps/)
- [MindTheGapps 16.0.0 arm64 releases](https://github.com/MindTheGapps/16.0.0-arm64/releases/latest)
- [Heimdall (Grimler fork, v2.2.x)](https://git.sr.ht/~grimler/Heimdall)
- [postmarketOS — Galaxy Tab S6 Lite (2020)](https://wiki.postmarketos.org/wiki/Samsung_Galaxy_Tab_S6_Lite_(2020)_(samsung-gta4xl))
- [GrapheneOS — sandboxed Google Play](https://grapheneos.org/usage#sandboxed-google-play)
- [Shelter](https://f-droid.org/packages/net.typeblog.shelter/)

---

## Licence

Documentation released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

No warranty of any kind. You are flashing your own hardware at your own risk.
