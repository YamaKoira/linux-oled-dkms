# intel-legacy-oled-dkms

An Arch Linux **PKGBUILD** and out-of-tree **i915** kernel module package that restores hardware brightness control on the **ThinkPad X1 Yoga 1st Gen (2016)** OLED display when running modern mainline Linux kernels.

Yes, this is made heavily using AI, but it's confirmed working on my machine at least. 

## 🌟 Background

On mainline Linux kernels, the 2016 Lenovo ThinkPad X1 Yoga (1st Gen) OLED screen lacks working hardware brightness control out of the box.

Chris Xiong ([Chrisoft](https://chrisoft.org/blog/post/2025-04-28.html)) reverse-engineered Lenovo's original debug driver and created a patch that ports the legacy DPCD backlight handling into the `i915` driver module.

## 📋 What This Package Does

When built and installed via `makepkg`, this package:

1. **Patches & Builds `i915`:** Downloads the corresponding Linux kernel source tree, applies the legacy OLED DPCD patch, fixes trace include paths for out-of-tree compilation, and builds the module via DKMS.
2. **Installs Display Calibration Firmware:** Places `intel_legacy_panel_data.bin` into `/usr/lib/firmware/` (containing gamma, ACL, and ELVSS tables required for DPCD communication). **This file is not shipped in this repository** — see below for how to obtain it.
3. **Configures `i915` Driver Options:** Drops `/etc/modprobe.d/i915-oled.conf` with `options i915 enable_dpcd_backlight=99` to force the `i915` driver to use the legacy OLED brightness path.

## 📥 Obtaining the Panel Data Blob

The `intel_legacy_panel_data.bin` file contains calibration tables extracted from the panel's original Windows debug driver. It is **not** distributed in this repository — you have to produce it yourself.

Follow the instructions in Chrisoft's write-up at <https://chrisoft.org/blog/post/2025-04-28.html> to extract it, then place the resulting file **in the same directory as the `PKGBUILD`** before running `makepkg`. The `PKGBUILD` expects it there and will fail with a source error otherwise.

> You are responsible for verifying that the data matches your panel revision.

## 📦 Installation (Arch Linux)

### Building from Source

```bash
# 1. Clone the repository
git clone https://gitlab.com/yamakoira/linux-oled-dkms.git
cd linux-oled-dkms

# 2. Follow the "Obtaining the Panel Data Blob" section above
#    and place intel_legacy_panel_data.bin next to PKGBUILD

# 3. Install your kernel headers, for example: 
sudo pacman -S linux-headers

# 4. Build and install the DKMS package
makepkg -si
```

### Post-Installation Setup

1. **Rebuild initramfs:** This is likely unncesessary because mkinitcpio should fire during install automatically, however, if `i915` is included in your `mkinitcpio.conf` `MODULES` list for early KMS, regenerate your initramfs:
   ```bash
   sudo mkinitcpio -P
   ```

2. **Reboot your system** to load the newly patched `i915` module and firmware.

## 🛠️ Verification

After rebooting, verify that the module parameter is correctly loaded:

```bash
cat /sys/module/i915/parameters/enable_dpcd_backlight
```

It should output **`99`**.

You can now adjust brightness natively using your laptop hotkeys, desktop environment brightness sliders, or utility tools like `brightnessctl` and `light`.

## 🏷️ Credits & References

* **Original Patch & Reverse Engineering:** [Chris Xiong (Chrisoft)](https://chrisoft.org/blog/post/2025-04-28.html)

## 📄 License
Probably GPL-2.0-only. No warranty, no guarantees, use at your own risk.

See individual source files for their respective licensing terms. 

:)
