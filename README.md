![AstralOS Logo](logo-small.png)
# AstralOS Manifest

Build AstralOS for **miatoll** (`curtana`, `joyeuse`, `excalibur`, `gram`) from source.

AstralOS = LineageOS 22.2 + AstralOS changes. This local manifest replaces the
16 LineageOS projects that carry AstralOS changes with their AstralOSProject
counterparts, and pulls in the miatoll device trees — so a plain
`repo init` + `repo sync` produces a complete, buildable tree.

## Prerequisites

- Linux (Debian/Ubuntu recommended), ~200 GB free disk, 16 GB+ RAM recommended
  (plus generous swap), and a `ccache` budget if you rebuild often
- LineageOS build packages — see the
  [LineageOS wiki build dependencies](https://wiki.lineageos.org/devices/#device-builds)
  for your distro
- `repo` launcher:
  ```bash
  mkdir -p ~/bin && curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
  chmod +x ~/bin/repo && export PATH=~/bin:$PATH
  ```

## Get the source

```bash
mkdir -p ~/astralos && cd ~/astralos

repo init -u https://github.com/LineageOS/android.git -b lineage-22.2

mkdir -p .repo/local_manifests
curl -fsSL -o .repo/local_manifests/zz-astral.xml \
  https://raw.githubusercontent.com/AstralOSProject/manifest/main/local_manifests/zz-astral.xml

repo sync -c -j"$(nproc)" --force-sync --no-clone-bundle --no-tags
```

The file name matters: it **must** be `zz-astral.xml`. repo loads local
manifests in sorted order, and `zz-` sorts after `roomservice.xml` (written by
`breakfast`), so the AstralOS project overrides always win.

## Build

```bash
source build/envsetup.sh
lunch lineage_miatoll-bp1a-userdebug
mka bacon
```

Output: `out/target/product/miatoll/AstralOS-<version>-OFFICIAL-miatoll.zip`

Verify the override took effect:

```bash
git -C frameworks/base remote get-url origin
# → https://github.com/AstralOSProject/frameworks_base.git
```

You do **not** need `breakfast` — the device trees are part of this manifest.
If you run it anyway, it may regenerate `.repo/local_manifests/roomservice.xml`;
that file loads before `zz-astral.xml`, so AstralOS repos still win. Re-run
`repo sync` afterwards if repo complains about duplicate paths.

## Other devices

miatoll is baked into this manifest; every other device needs its own tree
added (automatically via LineageOS roomservice if the device is official on
`lineage-22.2`, or with a small local-manifest snippet otherwise). Full
walkthrough with a worked example — **merlinx (Redmi Note 9)** — including
blob extraction and releasing:

**[docs/build-other-devices.md](docs/build-other-devices.md)**

## What this manifest changes

| Path | Source |
|---|---|
| `bootable/recovery` | AstralOSProject |
| `device/xiaomi/miatoll` | AstralOSProject |
| `device/xiaomi/sm6250-common` | AstralOSProject |
| `external/roboto-fonts` | AstralOSProject (Google Sans Flex) |
| `frameworks/base` | AstralOSProject (features, SystemUI, fonts) |
| `frameworks/native` | AstralOSProject (dalvik heap tuning) |
| `lineage-sdk` | AstralOSProject |
| `packages/apps/{Backgrounds,DocumentsUI,Jelly,LineageParts,Settings,SetupWizard,Trebuchet,Updater}` | AstralOSProject |
| `vendor/lineage` | AstralOSProject (branding) |
| `vendor/astralos` | AstralOSProject `build-scripts` (`update_base.sh`) |
| `hardware/sony/timekeep`, `hardware/xiaomi`, `kernel/xiaomi/sm6250` | LineageOS (unmodified) |

Everything else (~1130 projects) comes from LineageOS `lineage-22.2` unchanged.

## Updating the base

The LineageOS security base is refreshed periodically with
`vendor/astralos/update_base.sh` (core team). Rebase your device tree on the
updated base before shipping a release, and publish the build with a matching
OTA JSON in the [`ota`](https://github.com/AstralOSProject/ota) repository.

## Problems?

- `duplicate path` errors: make sure your override file is named
  `zz-astral.xml` and there is no older copy (e.g. `astral.xml`) left over.
- A project checked out from the wrong remote: `repo sync -c --force-sync <path>`.
- Anything else: open an issue on this repository or ask on
  [Discord](https://discord.gg/WCUbsQx3kE).

  ## Other

  - **You can try building AstralOS on the LineageOS 23.2 or 24.0 base**
