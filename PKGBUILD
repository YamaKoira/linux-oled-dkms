# Maintainer: YamaKoira
#
# Follows the installed kernel automatically:
#   makepkg -si                          # uses the installed 'linux' package version
#   KERNEL_PKG=linux-lts makepkg -si     # follow a different kernel package
#   KVER=7.2.15 makepkg -si              # force a specific kernel version
#
# One kernel flavour at a time: DKMS builds the same i915 source for every
# installed kernel, so keep this to the kernel package you actually run.

_kernel_version() {
    local raw
    if [[ -n ${KVER:-} ]]; then
        raw=$KVER
    else
        raw=$(pacman -Q "${KERNEL_PKG:-linux}" 2>/dev/null | awk '{print $2}')
        [[ -n $raw ]] || raw=$(uname -r)
    fi
    # 7.2.14.arch1-1 / 7.2.14-arch1-1 -> 7.2.14 ; 7.2.arch1-1 -> 7.2
    [[ $raw =~ ^([0-9]+\.[0-9]+(\.[0-9]+)?) ]] && printf '%s' "${BASH_REMATCH[1]}"
}

pkgname=intel-legacy-oled-dkms
pkgver=$(_kernel_version)
if [[ -z $pkgver ]]; then
    echo "Could not detect the kernel version, set KVER=x.y.z" >&2
    exit 1
fi
pkgrel=1
pkgdesc="Out-of-tree i915 DKMS patch for ThinkPad X1 Yoga 1st Gen OLED brightness (tracks the installed kernel)"
arch=('x86_64')
url="https://chrisoft.org/blog/post/2025-04-28.html"
# Kernel source is GPL-2.0-only, the added files are MIT (SPDX header in the patch).
# The license of intel_legacy_panel_data.bin is not covered by either.
license=('GPL-2.0-only' 'MIT')
depends=('dkms')
optdepends=('linux-headers: build against the linux kernel')
provides=('intel-legacy-oled')
conflicts=('intel-legacy-oled')

# kernel.org names the first release of a series linux-7.2.tar.xz, not 7.2.0
_srcver=$pkgver
[[ $pkgver =~ ^[0-9]+\.[0-9]+\.0$ ]] && _srcver=${pkgver%.0}

source=(
    "https://cdn.kernel.org/pub/linux/kernel/v${_srcver%%.*}.x/linux-${_srcver}.tar.xz"
    "intel_legacy_panel_data.bin"
    "i915-oled.conf"
    "dkms.conf"
    "linux-legacy-oled-brightness.patch"
)
sha256sums=('SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP')

prepare() {
    cd "$srcdir/linux-${_srcver}"

    if ! patch -Np1 --dry-run -i "$srcdir/linux-legacy-oled-brightness.patch" >/dev/null; then
        error "The OLED patch does not apply to linux-${_srcver}. It probably needs a rebase."
        return 1
    fi
    patch -Np1 -i "$srcdir/linux-legacy-oled-brightness.patch"

    # Fix TRACE_INCLUDE_PATH for out-of-tree DKMS builds.
    # define_trace.h resolves TRACE_INCLUDE_PATH relative to itself,
    # so the original "../../drivers/gpu/drm/i915/..." paths break in DKMS.
    find drivers/gpu/drm/i915 -maxdepth 1 -name '*trace*.h' -exec \
        sed -i 's|#define TRACE_INCLUDE_PATH ../../drivers/gpu/drm/i915.*|#define TRACE_INCLUDE_PATH .|' {} +

    sed -i 's|#define TRACE_INCLUDE_PATH \..*|#define TRACE_INCLUDE_PATH gvt|' \
        drivers/gpu/drm/i915/gvt/trace.h

    sed -i 's|#define TRACE_INCLUDE_PATH ../../drivers/gpu/drm/i915/display.*|#define TRACE_INCLUDE_PATH display|' \
        drivers/gpu/drm/i915/display/intel_display_trace.h

    # Warn if a future kernel adds a path the seds above did not catch
    if grep -rqE 'define TRACE_INCLUDE_PATH +\.\./' drivers/gpu/drm/i915; then
        warning "Some TRACE_INCLUDE_PATH lines are still relative to the kernel tree:"
        grep -rnE 'define TRACE_INCLUDE_PATH +\.\./' drivers/gpu/drm/i915 || true
    fi
}

package() {
    # 1. Firmware blob
    install -Dm644 "$srcdir/intel_legacy_panel_data.bin" \
        "$pkgdir/usr/lib/firmware/intel_legacy_panel_data.bin"

    # 2. Modprobe config (forces the legacy DPCD backlight path)
    install -Dm644 "$srcdir/i915-oled.conf" \
        "$pkgdir/etc/modprobe.d/i915-oled.conf"

    # 3. Stage DKMS source tree
    local dkmsdir="$pkgdir/usr/src/$pkgname-$pkgver"
    install -dm755 "$dkmsdir/drivers/gpu/drm"
    install -dm755 "$dkmsdir/drivers/platform"

    cp -r "$srcdir/linux-${_srcver}/drivers/gpu/drm/i915" "$dkmsdir/drivers/gpu/drm/"
    cp -r "$srcdir/linux-${_srcver}/drivers/platform/x86" "$dkmsdir/drivers/platform/"

    # 4. dkms.conf (version filled in from pkgver)
    install -Dm644 "$srcdir/dkms.conf" "$dkmsdir/dkms.conf"
    sed -i "s|@PKGVER@|$pkgver|" "$dkmsdir/dkms.conf"

    # 5. Kbuild shim for the i915 subdirectory
    cat << 'KBUILD' > "$dkmsdir/drivers/gpu/drm/i915/Kbuild"
ccflags-y += -I$(src) -I$(src)/display -I$(src)/gem -I$(src)/../../../../drivers/platform/x86
obj-m += i915.o
include $(src)/Makefile
KBUILD
}
