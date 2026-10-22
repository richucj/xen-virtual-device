# Android 17 (Linux 6.18) Xen Kernel Workspace Preparation and Build Guide

This guide documents the workspace preparation and build steps discussed
for the Xen Troops Android kernel configuration on Android 17 / Linux
6.18.

> **Current known state:** The Xen project is checked out at
> `common-modules/xen-virtual-device`, and Kleaf can query its targets.
> The distribution build currently stops during Bazel analysis because
> required external module packages---starting with
> `//modules/renesas/disfwk_fe:disfwk_fe`---are absent from the
> workspace. The exact repositories/manifest entries supplying those
> packages have not yet been identified. Do not treat the build as
> successful until those dependencies are resolved and the build
> completes.

## 1. Scope and assumptions

-   Host: Ubuntu Linux.
-   Workspace: `~/android17-kernel`.
-   Base manifest URL:
    `https://android.googlesource.com/kernel/manifest`.
-   Base manifest branch used when initializing the workspace:
    `common-android17-6.18`.
-   Xen project repository:
    `xen-troops/android_kernel_xen-virtual-device`.
-   Xen project revision: `common-android17-6.18-xenvm-trout-ih-main`.
-   Build system: Android Kleaf through Bazel.
-   Architecture/target: AArch64 (`xen_virtual_device_aarch64`).

This guide is for the Xen Troops Android 17 kernel workspace. It is not
a Raspberry Pi 5-specific guide. The Xen repository's `BUILD.bazel`
includes Renesas and Imagination external-module dependencies, so a
workspace intended to build this full target must provide the matching
source projects or otherwise use the correct upstream
manifest/configuration for the intended platform.

## 2. Install basic host tools

Install common tools needed to fetch and inspect the source:

``` bash
sudo apt update
sudo apt install -y git curl repo python3 python3-pip \
    build-essential bc bison flex libssl-dev \
    libelf-dev pahole rsync unzip zip xz-utils \
    zstd lz4 cpio file
```

The exact host packages required can vary with the kernel branch and
host image. If the build reports a missing host utility or library,
install the package indicated by the error and rerun the build.

Check the tools:

``` bash
git --version
repo --version
python3 --version
```

If the `repo` command is unavailable, install Google's Repo launcher
using the official Android developer instructions for your distribution.
Avoid downloading scripts from untrusted mirrors.

## 3. Initialize the Android kernel manifest

Create and enter the workspace:

``` bash
mkdir -p ~/android17-kernel
cd ~/android17-kernel
```

Initialize the Android 17 / Linux 6.18 kernel manifest:

``` bash
repo init \
  -u https://android.googlesource.com/kernel/manifest \
  -b common-android17-6.18
```

Sync the workspace:

``` bash
repo sync -j"$(nproc)"
```

This may take significant time and disk space. If the sync fails,
resolve the reported repository/network/authentication issue before
building.

Confirm that the workspace exists:

``` bash
pwd
ls -la
```

Expected high-level paths include `common/`, `build/` or `build/kernel`,
`common-modules/`, `prebuilts/`, and `.repo/`. The precise project list
is controlled by the manifest revision actually checked out.

## 4. Understand the active manifest

Inspect the manifest used by Repo:

``` bash
repo manifest -o /tmp/android17-kernel-manifest.xml
```

The workspace's `.repo/manifests/default.xml` may contain
project-specific revisions. Do not edit `.repo/manifests/default.xml`
merely to add a local project. Keep local additions in
`.repo/local_manifests/` so they are separate from the upstream
manifest.

In the manifest inspected during this work, examples included:

``` xml
<project path="build/kernel" name="kernel/build" groups="ddk">
  <linkfile src="kleaf/bazel.sh" dest="tools/bazel"/>
  <linkfile src="kleaf/bzlmod/bazel.MODULE.bazel" dest="MODULE.bazel"/>
</project>

<project path="common" name="kernel/common" revision="android17-6.18"/>

<project path="common-modules/virtual-device"
         name="kernel/common-modules/virtual-device"
         revision="android17-6.18"/>
```

The checked-in manifest may vary. Always inspect the actual manifest in
your workspace rather than assuming every project follows the same
revision.

## 5. Add the Xen Troops project through a local manifest

Create the local manifest directory:

``` bash
cd ~/android17-kernel
mkdir -p .repo/local_manifests
```

Create `.repo/local_manifests/xen.xml`:

``` bash
cat > .repo/local_manifests/xen.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote
      name="xen-troops"
      fetch="https://github.com/xen-troops/" />

  <project
      name="android_kernel_xen-virtual-device"
      path="common-modules/xen-virtual-device"
      remote="xen-troops"
      revision="common-android17-6.18-xenvm-trout-ih-main" />
</manifest>
EOF
```

**Important:** The repository name and path above match the Xen project
used in this workflow. If the upstream project changes or your team
provides a different manifest, use that authoritative source.

Sync only this project first:

``` bash
repo sync common-modules/xen-virtual-device
```

Then inspect it:

``` bash
ls -la common-modules/xen-virtual-device
```

Expected files include `BUILD.bazel`, `README.md`, and
fragment/configuration files such as `xenvm.aarch64.fragment`. The exact
list can change by revision.

## 6. Verify the Xen project revision

Run:

``` bash
cd ~/android17-kernel/common-modules/xen-virtual-device

git status
git log -1 --oneline
git rev-parse HEAD
git branch -a
git remote -v
```

During this workflow, the checkout reported:

``` text
220aa59 Revert "pcie6_rcar_gen5_host: temporaly disable"
220aa59516f185dbaee9c50853bb9bc085fe26d9
```

and showed the remote branch:

``` text
remotes/xen-troops/common-android17-6.18-xenvm-trout-ih-main
```

The commit hash is a checkpoint from this specific workspace, not a
guarantee that the remote branch will always point to the same commit.

`Not currently on any branch` is normal for many Repo-managed checkouts.
Repo often checks out a pinned commit in detached-HEAD state. Do not
manually switch branches unless you intentionally want to override the
manifest's revision.

Return to the workspace root:

``` bash
cd ~/android17-kernel
```

## 7. Verify Kleaf/Bazel is available

The manifest should provide `build/kernel` and create the `tools/bazel`
launcher through a manifest `linkfile`.

Check:

``` bash
ls -l tools/bazel
ls -ld build/kernel
```

Then:

``` bash
tools/bazel version
```

The exact Bazel version is selected by the checked-out Kleaf setup. Use
the workspace's `tools/bazel` launcher rather than an unrelated system
Bazel binary.

If `tools/bazel` is missing, inspect the active manifest and the
`build/kernel` project before proceeding:

``` bash
repo manifest -o /tmp/android17-kernel-manifest.xml
ls -la build/kernel
```

## 8. Query the Xen Kleaf targets

From the workspace root:

``` bash
cd ~/android17-kernel
tools/bazel query '//common-modules/xen-virtual-device:*'
```

The query in this workspace succeeded and listed targets including:

``` text
//common-modules/xen-virtual-device:xen_virtual_device_aarch64
//common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
//common-modules/xen-virtual-device:xen_virtual_device_aarch64_external_modules
```

It also listed outputs/targets for the kernel `Image`, `System.map`,
`.config`, and modules.

To inspect target declarations in the project:

``` bash
grep -nE 'name =|kernel_build|kernel_module_group|dist' \
  common-modules/xen-virtual-device/BUILD.bazel
```

To inspect the external module group:

``` bash
sed -n '250,330p' common-modules/xen-virtual-device/BUILD.bazel
```

The `xen_virtual_device_aarch64_dist` target is the distribution target.
The `xen_virtual_device_aarch64` target is the kernel build target.

## 9. Attempt the distribution build

From the workspace root, the command is:

``` bash
cd ~/android17-kernel

tools/bazel run \
  --verbose_failures \
  //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

For a kernel build without running the distribution target, use:

``` bash
tools/bazel build \
  //common-modules/xen-virtual-device:xen_virtual_device_aarch64
```

The kernel target can still depend on external module packages through
its declared dependency graph. If Bazel fails during package loading or
analysis, the kernel may not start compiling at all.

Do not assume that a successful `bazel query` means the build is ready.
Querying target names does not guarantee every dependency source tree is
present.

## 10. Current blocker: missing external module source packages

The distribution build currently fails with an error like:

``` text
ERROR: no such package 'modules/renesas/disfwk_fe':
BUILD file not found in any of the following directories.
```

The failing dependency is referenced by the Xen project's external
module group:

``` python
kernel_module_group(
    name = "xen_virtual_device_aarch64_external_modules",
    srcs = [
        "//common-modules/xen-virtual-device/dma_buf_cma_heap:dma_buf_cma_heap",
        "//modules/imagination/ddk/rogue_km:pvrsrvkm",
        "//modules/renesas/disfwk_fe:disfwk_fe",
        "//modules/renesas/rcar-mfis:rcar_mfis",
        "//modules/renesas/rcar_gen5_rproc:rcar_gen5_rproc",
        "//modules/renesas/pcie6_rcar_gen5_host:pcie6_rcar_gen5_host",
        "//modules/renesas/camfwk_fe:camfwk_fe",
        ...
    ],
)
```

This means the Xen project declares external module targets whose
package directories must exist and contain the expected Bazel BUILD
definitions. The current base workspace did not have at least
`modules/renesas/disfwk_fe`.

### 10.1 Check which module directories exist

``` bash
cd ~/android17-kernel

find modules -maxdepth 4 -type d 2>/dev/null \
  | grep -E 'renesas|imagination'
```

Check the specific missing directory:

``` bash
ls -ld modules/renesas/disfwk_fe
```

### 10.2 Search manifest definitions

``` bash
grep -R -nE \
  'disfwk_fe|rcar-mfis|rcar_gen5_rproc|pcie6_rcar_gen5_host|camfwk_fe|rogue_km' \
  .repo/manifests .repo/local_manifests 2>/dev/null
```

Also inspect the fully resolved manifest:

``` bash
repo manifest -o /tmp/android17-kernel-manifest.xml

grep -nE 'renesas|imagination|disfwk|rcar-mfis|rogue_km' \
  /tmp/android17-kernel-manifest.xml
```

### 10.3 Inspect the Xen project's README and dependency declarations

``` bash
sed -n '1,240p' common-modules/xen-virtual-device/README.md
sed -n '260,340p' common-modules/xen-virtual-device/BUILD.bazel
```

Search for references to the expected module repositories in the source
and local manifests. If these projects are hosted in a private or vendor
Git server, obtain the authoritative manifest or repository URLs from
the project owner/team. Do not guess repository URLs or branch names.

### 10.4 Resolve the missing dependencies correctly

There are two valid approaches, depending on the intended platform:

1.  **Build the complete intended platform configuration:** add the
    matching Renesas and Imagination projects, at compatible revisions,
    using the platform's authoritative manifest/local manifest.
2.  **Build a different platform configuration:** use the appropriate
    Xen/kernel branch and target for that platform, or make a deliberate
    and reviewed change to the external-module dependency list if those
    modules are genuinely not part of the target.

Do not create empty directories or empty `BUILD.bazel` files to silence
the error. That only masks the missing source and will fail later or
produce an incomplete distribution. Do not delete dependencies merely to
get a green build unless the platform owner confirms they are not
required.

After adding the correct source projects, sync them:

``` bash
repo sync <project-path-1> <project-path-2>
```

Replace the placeholders with the actual project paths from the
authoritative manifest.

Then re-run the target query and build:

``` bash
tools/bazel query '//common-modules/xen-virtual-device:*'

tools/bazel run \
  --verbose_failures \
  //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

## 11. Find the build outputs

Only after the build succeeds, inspect the output tree:

``` bash
find out -path '*xen_virtual_device_aarch64*' -type f | head -100
```

Find likely kernel artifacts:

``` bash
find out -type f \( -name Image -o -name System.map -o -name '*.ko' \) \
  | grep 'xen_virtual_device_aarch64' \
  | head -100
```

The target query showed output labels such as:

``` text
//common-modules/xen-virtual-device:xen_virtual_device_aarch64/Image
//common-modules/xen-virtual-device:xen_virtual_device_aarch64/arch/arm64/boot/Image
//common-modules/xen-virtual-device:xen_virtual_device_aarch64/System.map
```

These are Bazel labels, not literal filesystem paths. The physical
output location depends on the Kleaf/Bazel output configuration. For the
distribution target, inspect the successful build log and the target's
`BUILD.bazel` definitions to identify the final distribution directory.

Do not copy an artifact into a device image until you have confirmed
that it was produced by the successful build and is the correct artifact
for the intended platform.

## 12. Useful troubleshooting commands

### Check workspace state

``` bash
cd ~/android17-kernel
repo status
git -C common-modules/xen-virtual-device status
git -C common-modules/xen-virtual-device log -1 --oneline
```

### Check manifest and local manifest

``` bash
ls -la .repo/local_manifests
cat .repo/local_manifests/xen.xml
repo manifest -o /tmp/android17-kernel-manifest.xml
```

### Re-run the query

``` bash
tools/bazel query '//common-modules/xen-virtual-device:*'
```

### Show more detailed build errors

``` bash
tools/bazel run \
  --verbose_failures \
  --sandbox_debug \
  //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

Use `--sandbox_debug` when needed; it can produce more diagnostic
output.

### Clean only when necessary

Do not delete the whole `out/` directory as the first troubleshooting
step. First resolve missing source packages and configuration errors. If
a stale Bazel state is suspected, consult the Kleaf documentation for
the checked-out branch before using clean commands.

## 13. End-to-end command summary

Initial setup, if not already done:

``` bash
mkdir -p ~/android17-kernel
cd ~/android17-kernel

repo init \
  -u https://android.googlesource.com/kernel/manifest \
  -b common-android17-6.18

repo sync -j"$(nproc)"
```

Add the Xen project using `.repo/local_manifests/xen.xml` as described
in Section 5, then:

``` bash
repo sync common-modules/xen-virtual-device

cd ~/android17-kernel

tools/bazel version

tools/bazel query '//common-modules/xen-virtual-device:*'

tools/bazel run \
  --verbose_failures \
  //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

**Current status:** The Xen project checkout and target query have been
verified, but the distribution build is blocked by missing external
module packages. Resolve those dependencies from the correct platform
manifest before considering the kernel build complete.

## 14. Completion checklist

-   [ ] Host build tools installed.
-   [ ] Android 17 / Linux 6.18 kernel manifest initialized.
-   [ ] Base kernel repositories synced successfully.
-   [ ] Xen Troops project added through
    `.repo/local_manifests/xen.xml`.
-   [ ] Xen project revision verified.
-   [ ] `tools/bazel` launcher available.
-   [ ] Kleaf target query succeeds.
-   [ ] All external module repositories required by `BUILD.bazel` are
    present at compatible revisions.
-   [ ] `xen_virtual_device_aarch64_dist` completes successfully.
-   [ ] Distribution output directory and kernel `Image` verified.
-   [ ] Artifacts validated for the intended platform before deployment.