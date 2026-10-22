# android_kernel_xen-virtual-device

Xen **guest (domU)** kernel for aarch64, built with [Kleaf](https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/)
(the Bazel-based kernel build system used by Android/GKI). This configuration targets the
**Raspberry Pi 5** Xen guest (see [`xen-rpi5-guest.fragment`](xen-rpi5-guest.fragment)), but the
base `xen_virtual_device_aarch64` target is a generic aarch64 Xen virtual-device kernel.

---

## 1. Prerequisites

- A checked-out Android kernel manifest workspace (this repo is `common-modules/xen-virtual-device`,
  synced against `common-android17-6.18-xenvm-trout-ih-main`).
- The Bazel wrapper provided by the workspace: [`tools/bazel`](../../tools/bazel).
- A Linux host with the prebuilt Clang/LLVM toolchain that Kleaf downloads automatically
  (no system compiler required).

## 2. Build

From the workspace root:

```bash
tools/bazel build //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

To actually install the artifacts to a directory, use `run` instead of `build`:

```bash
tools/bazel run //common-modules/xen-virtual-device:xen_virtual_device_aarch64_dist
```

Use `--verbose_failures` to see the command lines of any failing step.

> A full build compiles the whole kernel + ~170 modules and takes roughly 10–20 minutes
> (depending on machine). Incremental rebuilds after a config-only change are much faster
> because the C compilation is cached.

## 3. Output locations

After a successful `build`, the artifacts live under the Bazel output tree
(`bazel-bin` → `bazel-out/k8-fastbuild/bin`):

| Artifact | Path |
| --- | --- |
| Kernel `Image` | `bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/arch/arm64/boot/Image` |
| In-tree modules (flat) | `bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/*.ko` |
| `lib/modules/<ver>/` tree | `bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64_modules_install/staging/lib/modules/<ver>/` |
| Final `.config` | `bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/.config` |

The kernel version is read from:

```bash
cat bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/include/config/kernel.release
```

### Copying artifacts out (example)

```bash
VER=$(cat bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/include/config/kernel.release)
DEST=~/xen-guest-artifacts
rm -rf "$DEST"; mkdir -p "$DEST/lib/modules"
cp bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/arch/arm64/boot/Image "$DEST/Image"
cp -a bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64_modules_install/staging/lib/modules/. "$DEST/lib/modules/"
```

## 4. Configuration fragments

The kernel config is assembled from `gki_defconfig` plus the
`post_defconfig_fragments` list in the [`xen_virtual_device_aarch64`](BUILD.bazel:254) rule,
applied in this order:

1. [`virtual_device_core.fragment`](virtual_device_core.fragment)
2. [`virtual_device.fragment`](virtual_device.fragment)
3. [`aarch64.fragment`](aarch64.fragment)
4. [`xenvm.aarch64.fragment`](xenvm.aarch64.fragment) — enables `CONFIG_XEN=y` and core Xen options
5. [`xen-rpi5-guest.fragment`](xen-rpi5-guest.fragment) — **RPi5 guest frontends** (see below)

### `xen-rpi5-guest.fragment`

Enables the paravirtualized frontend drivers a Xen **guest** needs:

- `CONFIG_XEN_BLKDEV_FRONTEND=y` — PV block device (xvda, …), kept as a fallback
- `CONFIG_XEN_NETDEV_FRONTEND=y` — PV network device
- `CONFIG_HVC_XEN=y` — Xen hypervisor console (note: the Kconfig symbol is `HVC_XEN`,
  **not** `CONFIG_XEN_HVC`)
- `CONFIG_XEN_XENBUS_FRONTEND=y` — xenbus frontend support
- `CONFIG_XEN_DOM0=n` — *intended* to disable the dom0/host path (see constraint below)
- `CONFIG_XEN_BACKEND=n` — frontend-only guest (disables `xen-pciback`)

It also enables the **Xen grant-based virtio transport** used for virtio-blk
backed by the `virtio-disk` userspace backend in DomD:

- `CONFIG_XEN_VIRTIO=y` — grant-DMA transport (see note below)
- `CONFIG_XEN_GNTDEV=y`, `CONFIG_XEN_GRANT_DEV_ALLOC=y`, `CONFIG_XEN_GRANT_DMA_ALLOC=y`
- `CONFIG_XEN_GRANT_DMA_OPS=y`, `CONFIG_XEN_GRANT_DMA_IOMMU=y` (selected by `XEN_VIRTIO`)
- `CONFIG_XEN_DEV_EVTCHN=y`, `CONFIG_XEN_FRONT_PGDIR_SHBUF=y`
- `CONFIG_VIRTIO=y`, `CONFIG_VIRTIO_MMIO=y`, `CONFIG_VIRTIO_BLK=y` — **built-in** so the
  block device is available during Android first-stage init

> **There is no `drivers/xen/xen-virtio.c` bus driver.** `CONFIG_XEN_VIRTIO` enables the
> grant-DMA layer ([`grant-dma-ops.c`](../../common/drivers/xen/grant-dma-ops.c) +
> [`grant-dma-iommu.c`](../../common/drivers/xen/grant-dma-iommu.c)); the device is probed by
> the standard `virtio_mmio` driver (`compatible = "virtio,mmio"`) with an
> `iommus = <&xen_grant_dma <backend-domid>>` property. See
> [`XEN_VIRTIO_BLK_CONTRACT.md`](XEN_VIRTIO_BLK_CONTRACT.md) for the exact DT node and
> xenstore keys the toolstack must generate.

These frontends are built **into** the `Image` (`=y`), so they do **not** appear as separate
`.ko` files — that is expected. Because `virtio_blk`, `virtio_mmio`, `xen-gntdev`,
`xen-gntalloc` and `xen-pciback` are now built-in (or disabled), they were removed from the
`module_outs` / `module_implicit_outs` lists in [`BUILD.bazel`](BUILD.bazel).

## 5. Known constraints / gotchas

### `CONFIG_XEN_DOM0` cannot be forced to `=n` on arm64

`config XEN_DOM0` is declared `def_bool y` (no prompt) in
[`common/arch/arm64/Kconfig`](../../common/arch/arm64/Kconfig:1685), so a fragment cannot
disable it. Kconfig keeps it `=y`. This is harmless for a guest: the reference working domU
kernel (`linux_pv_domu`) also ships `CONFIG_XEN_DOM0=y` and runs correctly as a guest. Do not
treat `CONFIG_XEN_DOM0=y` in the final `.config` as a build failure.

### External vendor modules are intentionally dropped

The `xen_virtual_device_aarch64_external_modules` group
([`BUILD.bazel`](BUILD.bazel:266)) references **only**
`//common-modules/xen-virtual-device/dma_buf_cma_heap:dma_buf_cma_heap`.

Previously it also listed Renesas (`disfwk_fe`, `rcar-mfis`, `rcar_gen5_rproc`,
`pcie6_rcar_gen5_host`, `camfwk_fe`) and Imagination (`pvrsrvkm`) modules under `//modules/...`.
Those packages are **not present** in this workspace, so they were removed to unblock the build.
If you need them back, sync the corresponding `//modules/...` projects into the workspace first.

### Declaring in-tree modules in `module_outs`

If the build fails with *"The following kernel modules are built but not copied"*, the kernel
config now builds extra in-tree modules that are not listed in `module_outs`. Add their
**basenames** to `_VIRT_COMMON_MODULES` / `_VIRT_AARCH64_MODULES`
([`BUILD.bazel`](BUILD.bazel:61)) and re-build. (The `coresight` modules were added this way.)
The error message even prints the exact `buildozer` command to run.

## 6. Verifying a build

```bash
# Valid ARM64 boot executable?
file bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/arch/arm64/boot/Image
# -> Linux kernel ARM64 boot executable Image, little-endian, 4K pages

# Required Xen guest flags actually landed?
grep -E '^CONFIG_XEN_(BLKDEV_FRONTEND|NETDEV_FRONTEND)=' \
  bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/.config
# -> CONFIG_XEN_BLKDEV_FRONTEND=y
# -> CONFIG_XEN_NETDEV_FRONTEND=y

# Xen grant-based virtio transport + built-in virtio-blk
grep -E '^CONFIG_(XEN_VIRTIO|VIRTIO_MMIO|VIRTIO_BLK)=' \
  bazel-bin/common-modules/xen-virtual-device/xen_virtual_device_aarch64/.config
# -> CONFIG_XEN_VIRTIO=y
# -> CONFIG_VIRTIO_MMIO=y
# -> CONFIG_VIRTIO_BLK=y
```

## 7. Repository layout

| File | Purpose |
| --- | --- |
| [`BUILD.bazel`](BUILD.bazel) | Kleaf targets: `kernel_build`, `kernel_module_group`, `kernel_modules_install`, `pkg_install` |
| [`xen-rpi5-guest.fragment`](xen-rpi5-guest.fragment) | RPi5 Xen guest frontend + Xen-virtio config fragment |
| [`XEN_VIRTIO_BLK_CONTRACT.md`](XEN_VIRTIO_BLK_CONTRACT.md) | Guest DT node + xenstore contract for grant-based virtio-blk |
| [`xenvm.aarch64.fragment`](xenvm.aarch64.fragment) | Base Xen (arm64) config fragment |
| [`virtual_device_core.fragment`](virtual_device_core.fragment) / [`virtual_device.fragment`](virtual_device.fragment) / [`aarch64.fragment`](aarch64.fragment) | Generic virtual-device / arch fragments |
| `dma_buf_cma_heap/` | The only out-of-tree module group built here |
| `modules.blocklist` | Module blocklist for the dist |
