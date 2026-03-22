# NVIDIA Open GPU Kernel Modules — BSB DSC Fix Patch Summary

All changes are against `open-gpu-kernel-modules` 580.119.02. Four files modified, three distinct fixes.

---

## Fix 1: DSC RC Parameter Tables — VESA DSC 1.1 Compliance

**File: `src/common/modeset/timing/nvt_dsc_pps.c`**

This is the primary fix that eliminates the block/tile "rainbow static" artifacts on the Bigscreen Beyond VR headset. NVIDIA's RC (Rate Control) parameter lookup tables for 8bpc / 8.0 bpp DSC deviate from the VESA DSC 1.1 specification in ranges 9-14. The `max_qp` values are too low, preventing the encoder from quantizing aggressively enough when the RC buffer is near overflow. This causes buffer overflow on high-detail content, corrupting the compressed output at block boundaries.

All values are corrected to match VESA DSC 1.1 Table E-5. Column index 4 corresponds to 8.0 bpp, which is the operating point used by the BSB.

```diff
--- a/src/common/modeset/timing/nvt_dsc_pps.c
+++ b/src/common/modeset/timing/nvt_dsc_pps.c
@@ -191,12 +191,12 @@ static const NvU8 minqp444_8b[15][37]={
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0}
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0}
        ,{ 5, 5, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0}
-       ,{ 6, 5, 5, 4, 4, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0}
+       ,{ 6, 5, 5, 4, 3, 4, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2, 2, 1, 1, 1, 1, 1, 1, 0, 0, 0}  // [9] col4: 4->3 (VESA DSC 1.1)
        ,{ 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 4, 4, 4, 4, 4, 3, 3, 3, 3, 2, 2, 1, 1, 1, 1, 1, 1, 1, 1, 0}
        ,{ 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 4, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 1, 1, 1, 1, 0}
        ,{ 6, 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 1, 1, 1, 1, 0}
-       ,{ 9, 9, 9, 9, 8, 8, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 1, 1, 1}
-       ,{14,14,13,13,12,12,12,12,11,11,10,10,10,10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 4, 3, 3, 3, 3}
+       ,{ 9, 9, 9, 9, 7, 8, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 1, 1, 1}  // [13] col4: 8->7 (VESA DSC 1.1)
+       ,{14,14,13,13,13,12,12,12,11,11,10,10,10,10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 4, 3, 3, 3, 3}  // [14] col4: 12->13 (VESA DSC 1.1)
 };

 static const NvU8 maxqp444_8b[15][37]={
@@ -210,11 +210,11 @@ static const NvU8 maxqp444_8b[15][37]={
        ,{10,10, 9, 9, 8, 8, 8, 8, 8, 8, 8, 8, 8, 7, 7, 6, 5, 5, 4, 4, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1}
        ,{11,11,10,10, 9, 9, 9, 9, 9, 9, 8, 8, 8, 7, 7, 6, 6, 5, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1}
        ,{12,11,11,10,10,10, 9, 9, 9, 9, 9, 9, 9, 8, 8, 7, 6, 6, 5, 5, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 1, 1, 1}
-       ,{12,12,11,11,10,10,10,10,10,10, 9, 9, 9, 8, 8, 7, 7, 6, 6, 6, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 1}
-       ,{12,12,12,11,11,11,10,10,10,10, 9, 9, 9, 9, 8, 8, 8, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 1}
-       ,{12,12,12,12,11,11,11,11,11,10,10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 1}
-       ,{13,13,13,13,12,12,11,11,11,11,10,10,10,10, 9, 9, 8, 8, 8, 8, 7, 7, 6, 6, 6, 6, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2}
-       ,{15,15,14,14,13,13,13,13,12,12,11,11,11,11,10,10,10, 9, 9, 9, 8, 8, 8, 8, 7, 7, 6, 6, 6, 6, 5, 5, 5, 4, 4, 4, 4}
+       ,{12,12,11,11,11,10,10,10,10,10, 9, 9, 9, 8, 8, 7, 7, 6, 6, 6, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 1}  // [10] col4: 10->11 (VESA DSC 1.1)
+       ,{12,12,12,11,12,11,10,10,10,10, 9, 9, 9, 9, 8, 8, 8, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 1}  // [11] col4: 11->12 (VESA DSC 1.1)
+       ,{12,12,12,12,13,11,11,11,11,10,10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 6, 6, 5, 5, 5, 5, 4, 4, 3, 3, 3, 3, 2, 2, 2, 2, 1}  // [12] col4: 11->13 (VESA DSC 1.1)
+       ,{13,13,13,13,13,12,11,11,11,11,10,10,10,10, 9, 9, 8, 8, 8, 8, 7, 7, 6, 6, 6, 6, 5, 5, 4, 4, 4, 4, 3, 3, 2, 2, 2}  // [13] col4: 12->13 (VESA DSC 1.1)
+       ,{15,15,14,14,15,13,13,13,12,12,11,11,11,11,10,10,10, 9, 9, 9, 8, 8, 8, 8, 7, 7, 6, 6, 6, 6, 5, 5, 5, 4, 4, 4, 4}  // [14] col4: 13->15 (VESA DSC 1.1)
 };
```

```diff
@@ -938,7 +938,7 @@ DSC_PpsCalcRcParam
             //else
             {
                 const NvU32 ofs_und6[] = { 0, -2, -2, -4, -6, -6, -8, -8, -8, -10, -10, -12, -12, -12, -12 };
-                const NvU32 ofs_und8[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -10, -12, -12, -12 };
+                const NvU32 ofs_und8[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -12, -12, -12, -12 };  // [11]: -10->-12 (VESA DSC 1.1)
                 const NvU32 ofs_und12[] = { 2, 0, 0, -2, -4, -6, -8, -8, -8, -10, -10, -10, -12, -12, -12 };
                 const NvU32 ofs_und15[] = { 10, 8, 6, 4, 2, 0, -2, -4, -6, -8, -10, -10, -12, -12, -12 };
```

**What changed (summary table):**

| Table | Range | Column | Old | New | Why |
|-------|-------|--------|-----|-----|-----|
| `minqp444_8b` | 9 | 4 | 4 | 3 | VESA min_qp for range 9 |
| `minqp444_8b` | 13 | 4 | 8 | 7 | VESA min_qp for range 13 |
| `minqp444_8b` | 14 | 4 | 12 | 13 | VESA min_qp for range 14 |
| `maxqp444_8b` | 10 | 4 | 10 | 11 | VESA max_qp for range 10 |
| `maxqp444_8b` | 11 | 4 | 11 | 12 | VESA max_qp for range 11 |
| `maxqp444_8b` | 12 | 4 | 11 | 13 | VESA max_qp for range 12 |
| `maxqp444_8b` | 13 | 4 | 12 | 13 | VESA max_qp for range 13 |
| `maxqp444_8b` | 14 | 4 | 13 | 15 | VESA max_qp for range 14 |
| `ofs_und8` | 11 | - | -10 | -12 | VESA bpg_offset for range 11 |

---

## Fix 2: flatnessDetThresh Undefined Behavior

**File: `src/nvidia-modeset/src/nvkms-evo4.c`** (Ada/Blackwell)

The DP DSC path computed `flatness_det_thresh` using `bitsPerPixelX16` (e.g. 129) instead of `bits_per_component` (e.g. 8). This produced `2 << 121`, which is undefined behavior for a 32-bit integer. On x86-64 the shift wraps to `2 << 25 = 0x4000000`, then truncates to 0 in the 10-bit register field. The fix extracts `bits_per_component` from PPS byte 3 bits [7:4]. The HDMI path was already correct. NVIDIA's own code had an `XXX` comment acknowledging this bug.

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

**File: `src/nvidia-modeset/src/nvkms-evo3.c`** (Turing/Ampere)

Identical fix applied to the Turing/Ampere HAL path. Same bug, same function signature, same fix.

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
+        NvU32 bpc = (pDscInfo->dp.pps[0] >> 28) & 0xF; // PPS byte 3 bits[7:4]
+        flatnessDetThresh = (bpc >= 8) ? (2 << (bpc - 8)) : 2;
+    }

     nvAssert((pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_DUAL) ||
                 (pDscInfo->dp.dscMode == NV_DSC_EVO_MODE_SINGLE));
```

---

## Fix 3: Bigscreen Beyond WAR Database Entry

**File: `src/common/displayport/src/dp_wardatabase.cpp`**

Adds the Bigscreen Beyond VR headset (ManufacturerID 0x2709, ProductID 0x1234) to the DisplayPort WAR database. The BSB connects through a link box that converts DP to optical. Forcing the maximum link configuration (4 lanes HBR2) ensures sufficient bandwidth and stable link training. Without this, the driver may negotiate a lower link config that is borderline for the BSB's 3840x1920@90Hz DSC mode.

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

## Environment

- GPU: NVIDIA RTX 5090 (Blackwell)
- Driver base: open-gpu-kernel-modules 580.119.02
- OS: Fedora 43, Linux 6.18.9
- Compositor: KDE Plasma (Wayland)
- Sink: Bigscreen Beyond VR headset (DSC 1.1, 8bpc, 8.0 bpp, RGB 4:4:4, 3840x1920@90Hz)
