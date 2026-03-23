# Building EmberHeart Kernel Locally

This guide walks you through compiling the EmberHeart kernel on your own Linux machine, replicating what the GitHub Actions CI does.

> [!NOTE]
> A **64-bit Linux** host (Ubuntu 22.04 LTS or later recommended) is required. At least **16 GB of RAM** and **~100 GB of free disk space** are recommended (the `repo sync` pulls kernel source, prebuilt toolchains, and all vendor repositories).

---

## 1. Install Dependencies

```bash
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  git curl ca-certificates build-essential clang lld flex bison \
  libelf-dev libssl-dev libncurses-dev zlib1g-dev liblz4-tool \
  libxml2-utils rsync unzip dwarves file python3 ccache
```

---

## 2. Install the `repo` Tool

```bash
sudo curl -s https://storage.googleapis.com/git-repo-downloads/repo \
  -o /usr/local/bin/repo
sudo chmod +x /usr/local/bin/repo
```

---

## 3. Configure Git (required by `repo`)

```bash
git config --global user.email "you@example.com"
git config --global user.name  "Your Name"
git config --global feature.manyFiles true
git config --global core.fsmonitor true
```

---

## 4. Sync the Kernel Source

Pick the Android / manifest combination you want to build. The examples below use **OOS16** (Android 15 / kernel 5.15) for the OnePlus 11.

| Manifest key | XML file |
|---|---|
| OOS14 | `oneplus_11_u.xml` |
| OOS15 | `oneplus_11_v.xml` |
| OOS16 | `oneplus_11_b.xml` |
| FAST\_BUILD\_OOS15 (trimmed) | *(URL – see below)* |

```bash
# Create a working directory
mkdir -p ~/emberheart-build && cd ~/emberheart-build

# Standard full manifest (e.g. OOS16)
MANIFEST="oneplus_11_b.xml"
BRANCH="oneplus/sm8550"

repo init \
  -u https://github.com/OnePlusOSS/kernel_manifest.git \
  -b "$BRANCH" \
  -m "$MANIFEST" \
  --repo-rev=v2.16 \
  --depth=1 \
  --no-clone-bundle \
  --no-tags

repo sync -c --no-clone-bundle --no-tags --optimized-fetch -j"$(nproc --all)"
```

> [!TIP]
> **Faster / trimmed sync (FAST\_BUILD\_OOS15)** – use a custom manifest URL to skip unnecessary history:
> ```bash
> MANIFEST_URL="https://raw.githubusercontent.com/nullptr-t-oss/kernel_patches/refs/heads/main/kernel_manifest/oneplus_11_v_kernel.xml"
> mkdir -p .repo/manifests
> curl --fail --location --proto '=https' "$MANIFEST_URL" -o .repo/manifests/temp_manifest.xml
> repo init \
>   -u https://github.com/OnePlusOSS/kernel_manifest.git \
>   -b oneplus/sm8550 \
>   -m temp_manifest.xml \
>   --repo-rev=v2.16 \
>   --depth=1 \
>   --no-clone-bundle \
>   --no-tags
> repo sync -c --no-clone-bundle --no-tags --optimized-fetch -j"$(nproc --all)"
> ```

---

## 5. Clone Build Dependencies

Run these commands from your working directory (`~/emberheart-build`):

```bash
# AnyKernel3 flasher
git clone --depth=1 https://github.com/TheWildJames/AnyKernel3.git -b gki-2.0

# General kernel patches (Sultan / WildKernels)
git clone --depth=1 https://github.com/TheWildJames/kernel_patches.git

# EmberHeart-specific patches
git clone --depth=1 https://github.com/nullptr-t-oss/kernel_patches.git my_patches

# SUSFS (determine correct branch from kernel version)
# For android13-5.15  → gki-android13-5.15
# For android14-6.1   → gki-android14-6.1
# For android15-6.6   → gki-android15-6.6
SUSFS_BRANCH="gki-android13-5.15"   # adjust to your target
git clone https://gitlab.com/simonpunk/susfs4ksu.git
cd susfs4ksu && git checkout "$SUSFS_BRANCH" && cd ..
```

---

## 6. Clean Up ABI Protected Exports

```bash
cd ~/emberheart-build/kernel_platform
rm -f common/android/abi_gki_protected_exports_* 2>/dev/null || true
rm -f msm-kernel/android/abi_gki_protected_exports_* 2>/dev/null || true
```

---

## 7. Add KernelSU-Next

```bash
cd ~/emberheart-build/kernel_platform

# Use the latest stable tag
curl --fail --location --proto '=https' \
  -LSs "https://raw.githubusercontent.com/KernelSU-Next/KernelSU-Next/dev/kernel/setup.sh" \
  | bash -

# Or pin to a specific branch / commit hash:
# curl ... | bash -s "dev"

git submodule update --init --recursive
```

---

## 8. Apply SUSFS Patches

```bash
cd ~/emberheart-build/kernel_platform

cp ../susfs4ksu/kernel_patches/50_add_susfs_in_gki-android13-5.15.patch ./common/
cp ../susfs4ksu/kernel_patches/fs/* ./common/fs/
cp ../susfs4ksu/kernel_patches/include/linux/* ./common/include/linux/

# Patch KernelSU-Next for SUSFS (version-dependent – example for v2.0.0)
cd ./KernelSU-Next
cp ../../susfs4ksu/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch ./
patch -p1 --forward < 10_enable_susfs_for_ksu.patch || true

# Apply any rejection fixes
for file in $(find ./kernel -maxdepth 2 -name "*.rej" -exec basename {} .rej \;); do
  patch -p1 --forward < "../../kernel_patches/next/susfs_fix_patches/v2.0.0/fix_${file}.patch"
done
patch -p1 --forward < "../../kernel_patches/next/susfs_fix_patches/v2.0.0/overwrite_hook_mode.patch"
patch -p1 --forward < "../../kernel_patches/next/susfs_fix_patches/v2.0.0/ksu_toolkit.patch"
cd ../

# Apply the main SUSFS patch to common
cd common
patch -p1 < 50_add_susfs_in_gki-android13-5.15.patch
cd ../
```

> [!NOTE]
> The exact patch filenames and fix scripts depend on the SUSFS version checked out. Check `susfs4ksu/kernel_patches/` for available files.

---

## 9. Apply EmberHeart & Performance Patches (OnePlus 11 / `kalama` SoC)

```bash
cd ~/emberheart-build/kernel_platform/common

# EmberHeart-specific patches
for patch_file in ../../../my_patches/kernel_patches/op11/common/*.patch; do
    echo "Applying: $patch_file"
    patch -p1 < "$patch_file"
done

# Performance patches from WildKernels
patch -p1 --forward < "../../../kernel_patches/common/mem_opt_prefetch.patch"
patch -p1 --forward < "../../../kernel_patches/common/minimise_wakeup_time.patch"
patch -p1 --forward < "../../../kernel_patches/common/int_sqrt.patch"
patch -p1 --forward < "../../../kernel_patches/common/force_tcp_nodelay.patch"
patch -p1 --forward < "../../../kernel_patches/common/reduce_gc_thread_sleep_time.patch"
patch -p1 --forward < "../../../kernel_patches/common/add_timeout_wakelocks_globally.patch"
patch -p1 --forward < "../../../kernel_patches/common/f2fs_reduce_congestion.patch"
patch -p1 --forward < "../../../kernel_patches/common/reduce_freeze_timeout.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/add_limitation_scaling_min_freq.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/re_write_limitation_scaling_min_freq.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/adjust_cpu_scan_order.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/avoid_extra_s2idle_wake_attempts.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/disable_cache_hot_buddy.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/f2fs_enlarge_min_fsync_blocks.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/increase_ext4_default_commit_age.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/increase_sk_mem_packets.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/reduce_pci_pme_wakeups.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/silence_irq_cpu_logspam.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/silence_system_logspam.patch"
patch -p1 -F3 --forward < "../../../kernel_patches/common/use_unlikely_wrap_cpufreq.patch"
```

---

## 10. Apply Kernel Config Flags

```bash
cd ~/emberheart-build/kernel_platform/common

# KernelSU-Next & SUSFS
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_KSU \
  --disable CONFIG_KSU_KPROBES_HOOK \
  --enable CONFIG_KSU_SUSFS \
  --enable CONFIG_KSU_SUSFS_HAS_MAGIC_MOUNT \
  --enable CONFIG_KSU_SUSFS_SUS_PATH \
  --enable CONFIG_KSU_SUSFS_SUS_MOUNT \
  --enable CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT \
  --enable CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT \
  --enable CONFIG_KSU_SUSFS_SUS_KSTAT \
  --disable CONFIG_KSU_SUSFS_SUS_OVERLAYFS \
  --enable CONFIG_KSU_SUSFS_TRY_UMOUNT \
  --enable CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT \
  --enable CONFIG_KSU_SUSFS_SPOOF_UNAME \
  --enable CONFIG_KSU_SUSFS_ENABLE_LOG \
  --enable CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS \
  --enable CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG \
  --enable CONFIG_KSU_SUSFS_OPEN_REDIRECT \
  --enable CONFIG_KSU_SUSFS_SUS_MAP \
  --disable CONFIG_KSU_SUSFS_SUS_SU \
  --enable CONFIG_TMPFS_XATTR \
  --enable CONFIG_TMPFS_POSIX_ACL

# BBR v1
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_TCP_CONG_ADVANCED \
  --enable CONFIG_TCP_CONG_BBR \
  --enable CONFIG_NET_SCH_FQ \
  --enable CONFIG_NET_SCH_FQ_CODEL

# Nethunter – Bluetooth HCI
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_BT_HCIBTUSB \
  --enable CONFIG_BT_HCIBTUSB_BCM \
  --enable CONFIG_BT_HCIBTUSB_RTL \
  --enable CONFIG_BT_HCIVHCI \
  --enable CONFIG_BT_HCIBCM203X \
  --enable CONFIG_BT_HCIBPA10X \
  --enable CONFIG_BT_HCIBFUSB

# Nethunter – AirSpy & HackRF
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_USB_AIRSPY \
  --enable CONFIG_USB_HACKRF

# Nethunter – USB Serial
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_USB_SERIAL \
  --enable CONFIG_USB_SERIAL_CONSOLE \
  --enable CONFIG_USB_SERIAL_GENERIC \
  --enable CONFIG_USB_SERIAL_CH341 \
  --enable CONFIG_USB_SERIAL_FTDI_SIO \
  --enable CONFIG_USB_SERIAL_PL2303

# TTL target
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_IP_NF_TARGET_TTL \
  --enable CONFIG_IP6_NF_TARGET_HL \
  --enable CONFIG_IP6_NF_MATCH_HL

# IP Set
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_IP_SET \
  --set-val CONFIG_IP_SET_MAX 65534 \
  --enable CONFIG_IP_SET_BITMAP_IP \
  --enable CONFIG_IP_SET_BITMAP_IPMAC \
  --enable CONFIG_IP_SET_BITMAP_PORT \
  --enable CONFIG_IP_SET_HASH_IP \
  --enable CONFIG_IP_SET_HASH_IPMARK \
  --enable CONFIG_IP_SET_HASH_IPPORT \
  --enable CONFIG_IP_SET_HASH_IPPORTIP \
  --enable CONFIG_IP_SET_HASH_IPPORTNET \
  --enable CONFIG_IP_SET_HASH_IPMAC \
  --enable CONFIG_IP_SET_HASH_MAC \
  --enable CONFIG_IP_SET_HASH_NETPORTNET \
  --enable CONFIG_IP_SET_HASH_NET \
  --enable CONFIG_IP_SET_HASH_NETNET \
  --enable CONFIG_IP_SET_HASH_NETPORT \
  --enable CONFIG_IP_SET_HASH_NETIFACE \
  --enable CONFIG_IP_SET_LIST_SET

# LRU Gen & build optimizations
# For O2 (default): keep CC_OPTIMIZE_FOR_PERFORMANCE enabled and CC_OPTIMIZE_FOR_PERFORMANCE_O3 disabled.
# For O3: swap the two --enable/--disable lines below and pass -O3 in KCFLAGS (step 11).
./scripts/config --file arch/arm64/configs/gki_defconfig \
  --enable CONFIG_LRU_GEN_ENABLED \
  --enable CONFIG_LTO_CLANG_THIN \
  --enable CONFIG_LTO_CLANG \
  --enable CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE \
  --disable CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE_O3 \
  --enable CONFIG_OPTIMIZE_INLINING \
  --set-val CONFIG_FRAME_WARN 0 \
  --enable CONFIG_TRACEPOINTS
```

---

## 11. Build the Kernel

```bash
cd ~/emberheart-build/kernel_platform/common

# Suppress local-version auto-detection
: > .scmversion

# Locate the bundled Clang toolchain
CLANG_BIN=$(ls -d ../prebuilts/clang/host/linux-x86/clang-r*/ 2>/dev/null \
            | sort -V | tail -n1)/bin

# Fall back to system clang if prebuilts are absent
if [ ! -x "${CLANG_BIN}/clang" ]; then
  CLANG_BIN="$(dirname "$(command -v clang)")"
fi

export PATH="${CLANG_BIN}:/usr/lib/ccache:$PATH"

export LLVM=1 LLVM_IAS=1
export ARCH=arm64 SUBARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-
export CROSS_COMPILE_COMPAT=arm-linux-androideabi-
export LD=ld.lld HOSTLD=ld.lld
export AR=llvm-ar NM=llvm-nm
export OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump STRIP=llvm-strip
export HOSTCC=clang HOSTCXX=clang++

# Generate base config
make O=out gki_defconfig

# Optional: set a custom kernel name string
scripts/config --file out/.config --set-str LOCALVERSION "-android13-EmberHeart"
scripts/config --file out/.config -d LOCALVERSION_AUTO

# Merge deferred config choices
make O=out olddefconfig

# Compile  (swap -O2 for -O3 here and enable CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE_O3 above for O3 optimization)
make -j"$(nproc --all)" O=out \
  KCFLAGS="-Wno-error -pipe -fno-stack-protector -O2" \
  KCPPFLAGS="-DCONFIG_OPTIMIZE_INLINING" \
  2>&1 | tee build.log

echo "Kernel image: $(realpath out/arch/arm64/boot/Image)"
```

---

## 12. Package with AnyKernel3

```bash
cd ~/emberheart-build

cp kernel_platform/common/out/arch/arm64/boot/Image AnyKernel3/Image

cd AnyKernel3
# Edit anykernel.sh line 7 to set the kernel string (optional)
sed -i '7 c\kernel.string=EmberHeart Kernel by nullptr-t-oss' anykernel.sh

zip -r ../EmberHeart_OP11_local.zip ./*
echo "Flashable zip: $(realpath ../EmberHeart_OP11_local.zip)"
```

Flash `EmberHeart_OP11_local.zip` using [Kernel Flasher](https://github.com/fatalcoder524/KernelFlasher/releases/latest) or any AnyKernel3-compatible recovery.

---

## 13. (Optional) Create a Patched Boot Image

If you prefer patching the stock boot image instead of using AnyKernel3:

```bash
cd ~/emberheart-build

# Download stock boot zip from the repo (or supply your own)
wget https://github.com/nullptr-t-oss/EmberHeart_OnePlus11/raw/refs/heads/main/tools/boot.zip
unzip boot.zip   # extracts e.g. boot.img

# Download magiskboot
wget https://github.com/magojohnji/magiskboot-linux/raw/refs/heads/main/x86_64/magiskboot
chmod +x magiskboot

IMAGE=kernel_platform/common/out/arch/arm64/boot/Image
BOOT=$(ls *.img | head -n1)

./magiskboot unpack "$BOOT"
cp -f "$IMAGE" kernel
./magiskboot repack "$BOOT"
mv new-boot.img EmberHeart_boot.img
echo "Patched boot image: $(realpath EmberHeart_boot.img)"
```

Flash `EmberHeart_boot.img` directly via fastboot:
```bash
fastboot flash boot EmberHeart_boot.img
```
