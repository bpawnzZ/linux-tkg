# ALPHA-BUILD — machine-specific 6.12 LTS kernel for LUCABRASI

Fork of `Frogging-Family/linux-tkg`. Built in GitHub Actions, never on the host.

## Why this configuration

| Decision | Reason |
|---|---|
| Base = **Linux 6.12 LTS** (`_version="6.12-latest"` → v6.12.112) | 6.12 is the **newest maintained series** nvidia **570.153.02** still builds against. |
| Not 6.16 (what we ran before) | **EOL** — absent from kernel.org's longterm table, and in tkg's own `_eol_kernels`. |
| Not 6.17 / 6.18 / 7.x | DKMS hard-fails: `nvidia/nv-dmabuf.c: implicit declaration of function 'dma_buf_attachment_is_dynamic'`. |
| `_processor_opt="skylake"` | i7-6820HK is Skylake-H → x86-64-v3 (AVX2/BMI2/FMA). Previous build was baseline `x86-64`. |
| Never `native`, never `x86-64-v4` | `native` tunes for the *CI runner's* CPU; v4 needs AVX-512, which this mobile part lacks. |
| `_configfile="config-alpha-6.12.txt"` | The host's own running config — inherits ZFS/nvidia/iwlwifi/docker/BTF/IBT/sched_ext as already proven. |
| `_kernel_on_diet=false`, `_modprobeddb=false` | Heuristic module stripping is what broke the previous build. |
| `_debugdisable=false` + `dwarves` | Keeps BTF, CET/IBT, sched_ext, lockup detectors. BTF needs `pahole`. |
| `_compiler=gcc`, `_lto_mode=no` | Matches the distro toolchain; tkg calls LTO "not recommended". |
| `_timer_freq=300` | What the machine runs today. The old build's 1000 Hz was the deviation. |
| `_custom_commandline=""` | ZFS root arrives from `zbm-kcl`; a built-in cmdline would be appended. |

## Layout

```
config-alpha-6.12.txt                  host's /proc/config.gz (base config)
.github/workflows/build-alpha-612.yml  workflow_dispatch button
.github/workflows/arch-builder.yml     all build knobs live in its env: block
ALPHA-BUILD.md                         this file
```

Upstream's six maintenance workflows (incl. `build-release-kernels.yml`, a cron
that builds the **latest** kernel across 12 combinations every 12 h) were
deleted from this fork so they cannot burn CI minutes or push commits.
Restore one if ever wanted: `git checkout upstream/master -- .github/workflows/<f>`.

## Build

```bash
gh workflow run build-alpha-612.yml --ref alpha/6.12-lts-eevdf
gh run watch                                        # or: gh run list --limit 5
gh run download <run-id> -n arch-kernel-packages-eevdf -D ~/kernel-out
```

Expect **1.5–3 h** on a 4-vCPU runner (cold, no ccache). The final step asserts
the config profile and fails the run if any symbol silently dropped.

## Install — verify BEFORE rebooting (Phase 2 gate)

```bash
cd ~/kernel-out
KREL=6.12.112-tkg-eevdf

# 1. headers first, then prove EVERY DKMS module builds against them
sudo pacman -U linux612-tkg-eevdf-headers-*.pkg.tar.zst
sudo dkms install nvidia/570.153.02 -k "$KREL"
sudo dkms install zfs/2.4.1        -k "$KREL"
sudo dkms install r8168/8.056.02   -k "$KREL"
sudo dkms install vboxhost/7.2.14_OSE -k "$KREL"
sudo dkms install i2c-ch341/20210501  -k "$KREL"
sudo dkms status                     # all must read "installed"

# 2. only then the kernel itself
sudo pacman -U linux612-tkg-eevdf-*.pkg.tar.zst

# 3. prove the initramfs carries the out-of-tree drivers
sudo lsinitcpio /boot/initramfs-"$KREL".img | grep -E 'nvidia\.ko|zfs\.ko'
```

If a `dkms install` fails, **stop** — do not install the kernel package. The
running system is untouched at that point.

## Boot / rollback

`linux-lts` and its ZFSBootMenu entry are never modified. `linux612-tkg-eevdf`
installs under its own pkgbase, so rollback = pick the other entry at the boot
menu. Nothing destructive happens until that selection.

## Measure — "laggy" must become a number

Run on both kernels and compare:

```bash
uname -r
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver \
    /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cyclictest -t8 -p80 -m -i200 -l20000        # worst-case scheduling latency
nvidia-smi; zpool status; zfs version
```

## Traps (learned the hard way)

- **A package name is not evidence.** The previous `-tkg-bore` kernel shipped
  **no `CONFIG_SCHED_BORE`** at all — the BORE patch never applied. Verify with
  `grep -c SCHED_BORE /boot/config-$(uname -r)` and `strings -a vmlinuz | grep -i bore`.
- **Env knobs must be real `customization.cfg` keys**, or they are filtered out
  silently. `_nofallback` (set by upstream CI) is inert. Check with
  `grep '^_knob=' customization.cfg`.
- **BTF needs `dwarves`** in the container or `CONFIG_DEBUG_INFO_BTF` drops.
- **`_kernel_on_diet` + `_debugdisable` + `_processor_opt="x86-64"`** were the
  upstream CI defaults that made the old build worse than stock.
- **`CONFIG_MSKYLAKE` does not exist in 6.12.** Its x86 CPU-family choice only
  honours `MK8/MPSC/MCORE2/MATOM/GENERIC_CPU`; the nearest (`MCORE2`) would
  inject `-march=core2` and could dilute `-march=skylake`. No config fragment
  is used — the ISA comes from `_processor_opt` alone.
