# intel-legacy-oled-dkms

An Arch Linux PKGBUILD that builds a patched out-of-tree i915 module, so brightness control works again on the OLED panel of the ThinkPad X1 Yoga 1st Gen (2016) with current mainline kernels.

Fair warning: a lot of this was written with AI help. It works on my machine, which is the only guarantee I can give.

## Background

On mainline Linux, the OLED screen in the 2016 X1 Yoga has no working hardware brightness control. Chris Xiong ([Chrisoft](https://chrisoft.org/blog/post/2025-04-28.html)) reverse-engineered Lenovo's original debug driver and wrote a patch that ports the legacy DPCD backlight handling into i915. This package just wraps his work so it's easy to install on Arch, and DKMS rebuilds the module automatically for kernel updates within the same series (see [After a kernel update](#after-a-kernel-update)).

## What it does

Running `makepkg` gives you a package that:

- downloads the kernel source matching your kernel, applies the OLED DPCD patch, fixes the trace include paths so it compiles out of tree, and builds the module through DKMS
- installs `intel_legacy_panel_data.bin` to `/usr/lib/firmware/` (the gamma, ACL and ELVSS tables the DPCD communication needs)
- adds `/etc/modprobe.d/i915-oled.conf` with `options i915 enable_dpcd_backlight=99`, which makes i915 use the legacy OLED brightness path

## The panel data blob

`intel_legacy_panel_data.bin` holds calibration tables pulled from the panel's original Windows debug driver. It isn't in this repo, so you'll need to extract it yourself. Chrisoft's write-up explains how: <https://chrisoft.org/blog/post/2025-04-28.html>

Put the resulting file in the same directory as the `PKGBUILD` before building, otherwise `makepkg` will stop with a source error. Also, make sure the data actually matches your panel revision. That part is on you.

## Installing

```bash
git clone https://gitlab.com/yamakoira/linux-oled-dkms.git
cd linux-oled-dkms

# put intel_legacy_panel_data.bin next to the PKGBUILD (see above)

# you need headers for your kernel, e.g.:
sudo pacman -S linux-headers

makepkg -si
```

mkinitcpio should run on its own during install, so you probably don't need to do anything else. But if you have `i915` in the `MODULES` array of your `mkinitcpio.conf` (early KMS), rebuild the initramfs to be safe:

```bash
sudo mkinitcpio -P
```

Then reboot so the patched module and firmware get loaded.

## Checking that it worked

```bash
cat /sys/module/i915/parameters/enable_dpcd_backlight
```

This should print `99`. After that, your brightness keys, desktop slider, `brightnessctl` or `light` should all work.

## After a kernel update

DKMS ships the i915 source from the kernel version you built the package against (say 7.2.14). On a patch update (7.2.14 to 7.2.15) it rebuilds that same source against the new headers, which normally just works.

On a new kernel series (7.2 to 7.3) the old source will most likely not build. i915 then falls back to the unpatched in-tree driver and brightness control disappears. Rebuild the package against the new kernel to fix it:

```bash
makepkg -si
```

The `PKGBUILD` stops early with a clear message if the patch no longer applies to the new kernel, or if the trace include paths need a new fix. In that case the patch needs a rebase.

## Credits

The patch and all the reverse engineering are by [Chris Xiong (Chrisoft)](https://chrisoft.org/blog/post/2025-04-28.html).

## License

Probably GPL-2.0-only. No warranty, use at your own risk. Individual source files carry their own licenses.

:)
