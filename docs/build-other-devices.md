# Building AstralOS for other devices

The [`zz-astral.xml`](../local_manifests/zz-astral.xml) manifest in this repo makes
`repo init` + `repo sync` produce a complete buildable tree for **miatoll**
(Redmi Note 9 Pro family). This guide covers every other device — using
**`merlinx` (Redmi Note 9)** as the worked example.

## Step 0 — set up the base tree

Same as the [README](../README.md): `repo init` LineageOS `lineage-22.2`, install
`zz-astral.xml`, `repo sync`. The AstralOS core (SystemUI, Settings, vendor/lineage
branding, fonts) is device-independent and works for any device.

## Step 1 — find out where your device stands

```bash
# Is the device an official LineageOS device? What does it need?
curl -fsSL https://download.lineageos.org/api/v2/devices/<codename> | python3 -m json.tool

# Which LineageOS branches does its device tree have?
git ls-remote --heads https://github.com/LineageOS/android_device_<vendor>_<codename> \
  "refs/heads/lineage-*"
```

**AstralOS is based on LineageOS 22.2 (Android 15, release `bp1a`).** Your device
tree must have a `lineage-22.2` branch (or a backport of one) to build unmodified.

You end up in one of two cases:

## Case A — the tree has a `lineage-22.2` branch: nothing to do

Example: `alioth` (POCO F3). The traditional LineageOS flow just works — do **not**
add anything to this manifest. Run:

```bash
source build/envsetup.sh
breakfast <codename>
```

What happens behind the scenes (`build/envsetup.sh` → `vendor/lineage/build/tools/roomservice.py`):

1. The lunch product doesn't exist locally yet, so roomservice scans LineageOS's
   GitHub repository list for `android_device_<vendor>_<codename>`.
2. It resolves the branch (default revision comes from the manifest:
   `lineage-22.2`), writes `.repo/local_manifests/roomservice.xml`, and `repo sync`s
   the tree.
3. It reads the tree's **`lineage.dependencies`** JSON file and repeats the process
   for every kernel/common/hardware repo listed there — recursively.

`breakfast <codename>` is a thin alias for `lunch lineage_<codename>-bp1a-userdebug`
(the `bp1a` release comes from `vendor/lineage/vars/aosp_target_release`).

AstralOS overrides always win: `zz-astral.xml` is loaded after `roomservice.xml`
(repo loads local manifests in sorted order), and `lunch`'s deps-only pass dedupes
against every local manifest, so it never fights our entries.

## Case B — no `lineage-22.2` branch: bring your own tree

### Worked example: merlinx (Redmi Note 9)

Real state of the world (checked October 2026):

| Item | Value |
|---|---|
| Official LineageOS? | Yes — currently ships **23.2** |
| Device tree | `LineageOS/android_device_xiaomi_merlinx` — branches: **`lineage-23.2` only** |
| Common tree | `android_device_xiaomi_mt6768-common` — **`lineage-23.2` only** |
| Kernel | `android_kernel_xiaomi_mt6768` — **`lineage-23.2` only** |
| Platform | MediaTek MT6768 (Helio G85) |
| Shared deps that still have `lineage-22.2` | `device/mediatek/sepolicy_vndr`, `hardware/mediatek`, `hardware/xiaomi` |

Dependency graph (from `lineage.dependencies` in each tree):

```
device/xiaomi/merlinx
└── device/xiaomi/mt6768-common
    ├── device/mediatek/sepolicy_vndr   (LineageOS, lineage-22.2 exists)
    ├── hardware/mediatek               (LineageOS, lineage-22.2 exists)
    ├── hardware/xiaomi                 (LineageOS, lineage-22.2 exists)
    └── kernel/xiaomi/mt6768            (lineage-23.2 only)
```

If you run `breakfast merlinx` as-is it fails: roomservice looks for the default
revision `lineage-22.2`, doesn't find it on the merlinx repos, and bails out with
`Repository for merlinx not found in the LineageOS Github repository list`.

Your options, in order of preference:

1. **Backport the tree (the normal maintainer job).** Fork the device tree, common
   tree, and kernel into AstralOSProject, then make them build against the 22.2
   platform — expect to revert Android-16-only pieces (AIDL HAL revisions,
   soong/release flags, sepolicy additions).
2. **Find a community tree that already supports 22.x.** Many devices have forks
   that kept the `lineage-22.2` branch after LineageOS moved on:

   ```bash
   # look at forks of the Lineage tree and check their branches
   git ls-remote --heads <fork-url> "refs/heads/lineage-22.2"
   ```
3. **`ROOMSERVICE_BRANCHES` escape hatch.** `ROOMSERVICE_BRANCHES=lineage-23.2
   breakfast merlinx` forces roomservice to check out the 23.2 tree. Useful for
   *reading* someone else's tree — but a 23.2 (Android 16) tree will not build
   against the 22.2 platform, so don't ship it.

### Declaring your trees

Keep LineageOS's repo naming for your forks (`android_device_xiaomi_merlinx`,
`android_kernel_xiaomi_mt6768`, …) and put them under AstralOSProject — roomservice
keys off that exact `android_device_*_<codename>` pattern to auto-resolve
`lineage.dependencies` later.

Then add `.repo/local_manifests/zz-merlinx.xml` (the `zz-` prefix is required so it
loads **after** `roomservice.xml`; see the README):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- AstralOS forks (branch main, remote defined in zz-astral.xml) -->
  <remove-project path="device/xiaomi/merlinx" optional="true" />
  <project path="device/xiaomi/merlinx" name="android_device_xiaomi_merlinx" remote="astral" />

  <remove-project path="device/xiaomi/mt6768-common" optional="true" />
  <project path="device/xiaomi/mt6768-common" name="android_device_xiaomi_mt6768-common" remote="astral" />

  <remove-project path="kernel/xiaomi/mt6768" optional="true" />
  <project path="kernel/xiaomi/mt6768" name="android_kernel_xiaomi_mt6768" remote="astral" />

  <!-- LineageOS shared deps, unchanged on lineage-22.2 (remote github = LineageOS) -->
  <remove-project path="device/mediatek/sepolicy_vndr" optional="true" />
  <project path="device/mediatek/sepolicy_vndr" name="LineageOS/android_device_mediatek_sepolicy_vndr" remote="github" />

  <remove-project path="hardware/mediatek" optional="true" />
  <project path="hardware/mediatek" name="LineageOS/android_hardware_mediatek" remote="github" />

  <!-- hardware/xiaomi is already shipped by zz-astral.xml — do NOT redefine it,
       or repo will fail with "duplicate path". -->
</manifest>
```

Notes:

- This snippet assumes `zz-astral.xml` is installed (it defines the `astral` remote).
- Every entry is `remove-project optional="true"` + re-add, so the file works even
  if `breakfast`/roomservice already cloned something at a different revision.
- `optional="true"` makes the remove a no-op when the path doesn't exist (fresh tree).
- Once your device builds, **send a PR** adding it to this repo's
  `local_manifests/zz-astral.xml` (or a new `zz-<device>.xml` here) so every builder
  gets it without manual steps, that's how miatoll ships.

```bash
repo sync -c -j"$(nproc)" --force-sync
```

## Step 2 — proprietary blobs (both cases)

Device trees ship only the *lists* (`proprietary-files.txt`); the actual blobs are
extracted locally and never committed:

```bash
cd device/xiaomi/<codename>
./extract-files.py                    # device connected over adb
./extract-files.py /path/to/dump      # or a directory with an extracted stock ROM
```

This creates `vendor/xiaomi/<codename>` (and common, if any) in your tree. For
merlinx, grab a Redmi Note 9 stock firmware, extract the images, and point the
script at that directory if you don't have the phone on hand.

## Step 3 — build

```bash
source build/envsetup.sh
breakfast <codename>      # or: lunch lineage_<codename>-bp1a-userdebug
mka bacon
```

Output: `out/target/product/<codename>/AstralOS-<version>-OFFICIAL-<codename>.zip`

Confirm it's really AstralOS:

```bash
grep ro.build.display.id out/target/product/<codename>/system/build.prop
# ro.build.display.id=AstralOS-<version>-OFFICIAL-<codename>
```

## Step 4 — release

1. **Test the build on real hardware (boot, telephony, Wi-Fi, Bluetooth, camera.**
2. **Ask the owner to upload the zip as a release asset (same pattern as miatoll:
   `https://github.com/AstralOSProject/ota/releases/download/<tag>/<zip>`).**
3. **Ask the owner to add `https://astralosproject.github.io/ota/<codename>.json` to the
   [`ota`](https://github.com/AstralOSProject/ota) repo — schema in its README.**
4. **Ask on [Discord](https://discord.gg/WCUbsQx3kE) before calling the build OFFICIAL!**
