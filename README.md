# Bigscreen Beyond DSC Fix — NVIDIA Open GPU Kernel Modules Patch

Fixes Display Stream Compression (DSC) "rainbow static" block artifacts on the Bigscreen Beyond (BSB) VR headset when using NVIDIA open GPU kernel modules on Linux.

This patch contains three fixes and one convenience change against `open-gpu-kernel-modules` 580.119.02.

---

## WARNINGS

### Back up your working driver BEFORE applying this patch

```bash
# Create a backup of your current working nvidia modules
sudo mkdir -p /lib/modules/$(uname -r)/extra/nvidia-backup
sudo cp /lib/modules/$(uname -r)/extra/nvidia/*.ko /lib/modules/$(uname -r)/extra/nvidia-backup/
```

### Ensure you have SSH access or a recovery method

If the patched driver causes a black screen, broken display, or boot failure, you will need an alternative way to access your machine:

- **SSH**: Ensure `sshd` is running and you know your machine's IP. Test SSH access BEFORE rebooting with the patched driver.
  ```bash
  sudo systemctl enable sshd
  sudo systemctl start sshd
  ip addr show  # note your IP
  ```
- **TTY console**: You can switch to a text console with `Ctrl+Alt+F2` (may not work if the GPU driver is completely broken).
- **Boot to older kernel**: If you have multiple kernels in GRUB, you can boot an older one where the stock driver is still installed.
- **Single-user mode**: At the GRUB menu, press `e`, add `3` or `systemd.unit=multi-user.target` to the kernel line, press `Ctrl+X`. This boots without a graphical desktop.
- **Live USB**: Keep a Fedora/Ubuntu live USB handy as a last resort.

### Disable automatic kernel module rebuilds during testing

If you use `akmods` (Fedora) or `dkms`, disable automatic rebuilds so a kernel update doesn't overwrite your patched modules:

```bash
# Fedora (akmods)
sudo systemctl stop akmods.service
sudo systemctl disable akmods.service

# Re-enable when done testing
sudo systemctl enable akmods.service
sudo systemctl start akmods.service
```

---

## What this patch fixes

### Fix 1: DSC RC Parameter Tables (PRIMARY FIX)

**File:** `src/common/modeset/timing/nvt_dsc_pps.c`

This is the fix that eliminates the block/tile "rainbow static" artifacts. NVIDIA's RC (Rate Control) parameter lookup tables for 8bpc / 8.0 bpp DSC deviate from the VESA DSC 1.1 specification in ranges 9-14. The `max_qp` values are too constrained, preventing the encoder from quantizing aggressively enough when the RC buffer approaches overflow on high-detail content. This causes the RC buffer to overflow, corrupting the compressed output at DSC block boundaries — visible as "rainbow static" artifacts.

All values are corrected to match VESA DSC 1.1 Table E-5. Column index 4 corresponds to 8.0 bpp, which is the operating point used by the BSB.

**`minqp444_8b` table — min quantization parameter corrections:**

```diff
--- a/src/common/modeset/timing/nvt_dsc_pps.c
+++ b/src/common/modeset/timing/nvt_dsc_pps.c
@@ -191,12 +191,12 @@ static const NvU8 minqp444_8b[15][37]={
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0}
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0}
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0}
-       ,{ 6, 5, 5, 4, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0}
+       ,{ 6, 5, 5, 4, 3, 4, 3, 3, ...}  // [9] col4: 4->3 (VESA DSC 1.1)
        ,{ 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 4, 4, 4, 4, 4, 3, 3, 3, 3, 2, 2, 1, 1, 1, 1, 1, 1, 1, 1, 0}
        ,{ 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 4, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 1, 1, 1, 1, 0}
        ,{ 6, 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 1, 1, 1, 1, 0}
-       ,{ 9, 9, 9, 9, 8, 8, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 1, 1, 1}
-       ,{14,14,13,13,12,12,12,12,11,11,10,10,10,10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 4, 3, 3, 3, 3}
+       ,{ 9, 9, 9, 9, 7, 8, 7, 7, ...}  // [13] col4: 8->7 (VESA DSC 1.1)
+       ,{14,14,13,13,13,12,12,12, ...}  // [14] col4: 12->13 (VESA DSC 1.1)
 };
```

**`maxqp444_8b` table — max quantization parameter corrections (most critical):**

```diff
@@ -210,11 +210,11 @@ static const NvU8 maxqp444_8b[15][37]={
        ,{10,10, 9, 9, 8, 8, 8, 8, 8, 8, 8, 8, 8, 7, 7, 6, 5, 5, 4, 4, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1}
        ,{11,11,10,10, 9, 9, 9, 9, 9, 9, 8, 8, 8, 7, 7, 6, 6, 5, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1}
        ,{12,11,11,10,10,10, 9, 9, 9, 9, 9, 9, 9, 8, 8, 7, 6, 6, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1}
-       ,{12,12,11,11,10,10,10,10, ...}  // [10] old max_qp=10
-       ,{12,12,12,11,11,11,10,10, ...}  // [11] old max_qp=11
-       ,{12,12,12,12,11,11,11,11, ...}  // [12] old max_qp=11
-       ,{13,13,13,13,12,12,11,11, ...}  // [13] old max_qp=12
-       ,{15,15,14,14,13,13,13,13, ...}  // [14] old max_qp=13
+       ,{12,12,11,11,11,10,10,10, ...}  // [10] col4: 10->11 (VESA DSC 1.1)
+       ,{12,12,12,11,12,11,10,10, ...}  // [11] col4: 11->12 (VESA DSC 1.1)
+       ,{12,12,12,12,13,11,11,11, ...}  // [12] col4: 11->13 (VESA DSC 1.1)
+       ,{13,13,13,13,13,12,11,11, ...}  // [13] col4: 12->13 (VESA DSC 1.1)
+       ,{15,15,14,14,15,13,13,13, ...}  // [14] col4: 13->15 (VESA DSC 1.1)
 };
```

**`ofs_und8` — bpg_offset correction:**

```diff
@@ -938,7 +938,7 @@ DSC_PpsCalcRcParam
             {
                 const NvU32 ofs_und6[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -12, -12, -12, -12 };
-                const NvU32 ofs_und8[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -10, -12, -12, -12 };
+                const NvU32 ofs_und8[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -12, -12, -12, -12 };
                 const NvU32 ofs_und12[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -10, -12, -12, -12 };
```

**Summary of all RC parameter changes:**

| Table | Range | Column | Old | New | VESA DSC 1.1 Reference |
|-------|-------|--------|-----|-----|------------------------|
| `minqp444_8b` | 9 | 4 | min_qp=4 | min_qp=3 | Table E-5, range 9 |
| `minqp444_8b` | 13 | 4 | min_qp=8 | min_qp=7 | Table E-5, range 13 |
| `minqp444_8b` | 14 | 4 | min_qp=12 | min_qp=13 | Table E-5, range 14 |
| `maxqp444_8b` | 10 | 4 | max_qp=10 | max_qp=11 | Table E-5, range 10 |
| `maxqp444_8b` | 11 | 4 | max_qp=11 | max_qp=12 | Table E-5, range 11 |
| `maxqp444_8b` | 12 | 4 | max_qp=11 | max_qp=13 | Table E-5, range 12 |
| `maxqp444_8b` | 13 | 4 | max_qp=12 | max_qp=13 | Table E-5, range 13 |
| `maxqp444_8b` | 14 | 4 | max_qp=13 | max_qp=15 | Table E-5, range 14 |
| `ofs_und8` | 11 | — | bpg_offset=-10 | bpg_offset=-12 | Table E-5, range 11 |

**Why this matters:** The max_qp values control how aggressively the DSC encoder can quantize when the RC buffer is filling up. With the old (too-low) values, the encoder runs out of quantization headroom on high-detail content, the RC buffer overflows, and the compressed bitstream becomes corrupted at block boundaries. Correcting these to VESA spec values gives the encoder the full quantization range it needs.

---

### Fix 2: flatnessDetThresh Undefined Behavior

**Files:** `src/nvidia-modeset/src/nvkms-evo4.c` (Ada/Blackwell), `src/nvidia-modeset/src/nvkms-evo3.c` (Turing/Ampere)

The DP DSC path computed `flatness_det_thresh` using `bitsPerPixelX16` (e.g. 129) instead of `bits_per_component` (e.g. 8). This produced `2 << 121`, which is undefined behavior for a 32-bit integer. On x86-64 this wraps to `2 << 25 = 0x4000000`, then truncates to 0 in the 10-bit register field. The fix extracts `bits_per_component` from PPS byte 3. NVIDIA's own code had an `XXX` comment acknowledging this bug. The HDMI DSC path was already correct.

```diff
--- a/src/nvidia-modeset/src/nvkms-evo4.c
+++ b/src/nvidia-modeset/src/nvkms-evo4.c
@@ -1409,10 +1409,13 @@ static void EvoSetDpDscParamsC9(const NVDispEvoRec *pDispEvo,

     nvAssert(pDscInfo->type == NV_DSC_INFO_EVO_TYPE_DP);

-    // XXX: I'm pretty sure that this is wrong.
-    // BitsPerPixelx16 is something like (24 * 16) = 384, and 2 << (384 - 8) is
-    // an insanely large number.
-    flatnessDetThresh = (2 << (pDscInfo->dp.bitsPerPixelX16 - 8)); /* ??? */
+    // Fix: use bits_per_component from PPS byte 3 [7:4], not bitsPerPixelX16.
+    // DSC spec: flatness_det_thresh = 2 << (bpc - 8).
+    // PPS DW[0] packs bytes [3][2][1][0] as [31:24][23:16][15:8][7:0].
+    {
+        NvU32 bpc = (pDscInfo->dp.pps[0] >> 28) & 0xF;
+        flatnessDetThresh = (bpc >= 8) ? (2 << (bpc - 8)) : 2;
+    }

     nvAssert((pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_DUAL) ||
                 (pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_SINGLE));
```

The identical fix is applied to `nvkms-evo3.c` for the Turing/Ampere HAL path:

```diff
--- a/src/nvidia-modeset/src/nvkms-evo3.c
+++ b/src/nvidia-modeset/src/nvkms-evo3.c
@@ -7816,10 +7816,13 @@ static void EvoSetDpDscParams(const NVDispEvoRec *pDispEvo,

     nvAssert(pDscInfo->type == NV_DSC_INFO_EVO_TYPE_DP);

-    // XXX: I'm pretty sure that this is wrong.
-    // BitsPerPixelx16 is something like (24 * 16) = 384, and 2 << (384 - 8) is
-    // an insanely large number.
-    flatnessDetThresh = (2 << (pDscInfo->dp.bitsPerPixelX16 - 8)); /* ??? */
+    // Fix: use bits_per_component from PPS byte 3 [7:4], not bitsPerPixelX16.
+    // DSC spec: flatness_det_thresh = 2 << (bpc - 8).
+    // PPS DW[0] packs bytes [3][2][1][0] as [31:24][23:16][15:8][7:0].
+    {
+        NvU32 bpc = (pDscInfo->dp.pps[0] >> 28) & 0xF;
+        flatnessDetThresh = (bpc >= 8) ? (2 << (bpc - 8)) : 2;
+    }

     nvAssert((pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_DUAL) ||
                 (pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_SINGLE));
```

---

### Fix 3: Bigscreen Beyond WAR Database Entry

**File:** `src/common/displayport/src/dp_wardatabase.cpp`

Adds the BSB (ManufacturerID 0x2709, ProductID 0x1234) to the DisplayPort workaround database. Forces maximum link configuration (4 lanes HBR2) to ensure sufficient bandwidth through the BSB's DP-to-optical link box.

```diff
--- a/src/common/displayport/src/dp_wardatabase.cpp
+++ b/src/common/displayport/src/dp_wardatabase.cpp
@@ -542,6 +542,19 @@ void Edid::applyEdidWorkArounds(NvU32 warFlag, const DpMonitorDenylistData *pDen
             }
             break;

+        // Bigscreen Beyond VR headset
+        case 0x2709:
+            if (ProductID == 0x1234)
+            {
+                //
+                // The Bigscreen Beyond connects via a link box (DP -> optical).
+                // Force 4 lanes HBR2 to ensure sufficient bandwidth.
+                //
+                this->WARFlags.forceMaxLinkConfig = true;
+                DP_PRINTF(DP_NOTICE, "DP-WAR> Force maximum link config for Bigscreen Beyond VR headset.");
+            }
+            break;
+
         // CMN
         case 0xAE0D:
             if (ProductID == 0x1747)
```

---

### Convenience: DRM Modeset Default

**Files:** `kernel-open/nvidia-drm/nvidia-drm-linux.c`, `kernel-open/nvidia-drm/nvidia-drm-os-interface.c`

Changes the `nvidia-drm.modeset` kernel parameter default from `false` to `true`. Required for Wayland compositors. Without this you must pass `nvidia-drm.modeset=1` on the kernel command line. **You may omit this change if you already set this parameter via modprobe config or kernel cmdline.**

```diff
--- a/kernel-open/nvidia-drm/nvidia-drm-os-interface.c
+++ b/kernel-open/nvidia-drm/nvidia-drm-os-interface.c
@@ -38,7 +38,7 @@
-bool nv_drm_modeset_module_param = false;
+bool nv_drm_modeset_module_param = true;
```

```diff
--- a/kernel-open/nvidia-drm/nvidia-drm-linux.c
+++ b/kernel-open/nvidia-drm/nvidia-drm-linux.c
@@ -31,7 +31,7 @@
 MODULE_PARM_DESC(
     modeset,
-    "Enable atomic kernel modesetting (1 = enable, 0 = disable (default))");
+    "Enable atomic kernel modesetting (1 = enable (default), 0 = disable)");
```

---

## Prerequisites

- NVIDIA open-gpu-kernel-modules source matching your installed driver version (tested: 580.119.02)
- Kernel headers for your running kernel
- Build tools: `gcc`, `make`
- Root access

```bash
# Fedora
sudo dnf install kernel-devel kernel-headers gcc make

# Ubuntu/Debian
sudo apt install linux-headers-$(uname -r) build-essential
```

---

## Installation

### Step 1: Clone and patch the source

```bash
# Clone the source (use the tag matching your driver version)
git clone --branch 580.119.02 --depth 1 \
    https://github.com/NVIDIA/open-gpu-kernel-modules.git
cd open-gpu-kernel-modules

# Apply the patch
git apply /path/to/bsb-dsc-fix.patch
```

### Step 2: Build

```bash
make modules -j$(nproc)
```

Build should complete with only warnings (objtool `.eh_frame` warnings and `missing MODULE_DESCRIPTION` are expected and harmless).

### Step 3: Back up existing modules

```bash
sudo mkdir -p /lib/modules/$(uname -r)/extra/nvidia-backup
sudo cp /lib/modules/$(uname -r)/extra/nvidia/*.ko \
        /lib/modules/$(uname -r)/extra/nvidia-backup/
```

### Step 4: Install patched modules

```bash
sudo cp kernel-open/nvidia.ko         /lib/modules/$(uname -r)/extra/nvidia/
sudo cp kernel-open/nvidia-modeset.ko  /lib/modules/$(uname -r)/extra/nvidia/
sudo cp kernel-open/nvidia-drm.ko     /lib/modules/$(uname -r)/extra/nvidia/
sudo cp kernel-open/nvidia-uvm.ko     /lib/modules/$(uname -r)/extra/nvidia/
sudo depmod -a
```

### Step 5: Reboot

```bash
sudo reboot
```

### Step 6: Verify

After reboot, check that the patched modules loaded:

```bash
# Check module is loaded
lsmod | grep nvidia

# Check dmesg for BSB WAR message (only if BSB is connected)
dmesg | grep "DP-WAR"
# Expected: "DP-WAR> Force maximum link config for Bigscreen Beyond VR headset."
```

---

## Recovery: Restoring the original driver

If the patched driver causes problems (black screen, artifacts, failure to boot to desktop):

### Option A: Via SSH

```bash
ssh user@your-machine-ip

# Restore backup
sudo cp /lib/modules/$(uname -r)/extra/nvidia-backup/*.ko \
        /lib/modules/$(uname -r)/extra/nvidia/
sudo depmod -a
sudo reboot
```

### Option B: Via TTY console

Press `Ctrl+Alt+F2` to get a text login, then run the same commands as Option A.

### Option C: Via single-user mode

1. Reboot, hold `Shift` (BIOS) or `Esc` (UEFI) to get the GRUB menu
2. Press `e` on your kernel entry
3. Find the line starting with `linux` and append: `3`
4. Press `Ctrl+X` to boot
5. Log in at the text prompt and restore the backup as above

### Option D: Via live USB

1. Boot from a Fedora/Ubuntu live USB
2. Mount your root partition:
   ```bash
   sudo mount /dev/your-root-partition /mnt
   ```
3. Restore the backup:
   ```bash
   sudo cp /mnt/lib/modules/$(ls /mnt/lib/modules/)/extra/nvidia-backup/*.ko \
           /mnt/lib/modules/$(ls /mnt/lib/modules/)/extra/nvidia/
   ```
4. Reboot into your installed system

---

## Environment tested

- GPU: NVIDIA RTX 5090 (Blackwell, GB202)
- Driver: open-gpu-kernel-modules 580.119.02
- OS: Fedora 43
- Kernel: 6.18.9-200.fc43.x86_64
- Desktop: KDE Plasma 6 (Wayland)
- VR Runtime: SteamVR
- Sink: Bigscreen Beyond VR headset (DSC 1.1, 8bpc, 8.0 bpp, RGB 4:4:4, 3840x1920@90Hz)

## Result

- **Before patch**: Persistent block-aligned "rainbow static" artifacts on high-detail VR content
- **After patch**: Artifacts completely eliminated; display output matches Windows quality
