# Maintainer: mklnln <mklnln@users.noreply.github.com>
# Unofficial package: Wine 11.0 with giang17's Direct2D / DirectComposition patch
# series (branch d2d1-dcomp-11.0), which makes JUCE8 / VSTGUI plugin GUIs
# (Pianoteq 9, Serum 2, Korg, etc.) render correctly under Wine.
# Installs isolated under /opt/wine-d2d1 so it does NOT replace system wine.
# Patch source: https://github.com/giang17/wine  (branch d2d1-dcomp-11.0)
# Inspired by patrickl's Fedora COPR wine-11.8-vstgui-juce8.

pkgname=wine-d2d1-dcomp-git
pkgver=11.0
pkgrel=1
pkgdesc="Wine with giang17 Direct2D/DComp patches for JUCE8/VSTGUI plugins (Pianoteq 9 etc.), installed to /opt/wine-d2d1"
arch=('x86_64')
url="https://github.com/giang17/wine"
license=('LGPL-2.1-or-later')
# Runtime + build deps mirror a functional Wine build (audio, fonts, vulkan for D2D path).
# new-WoW64 build (--enable-archs=i386,x86_64) means NO lib32-* deps are needed.
depends=('freetype2' 'fontconfig' 'libpng' 'libjpeg-turbo' 'giflib' 'gnutls'
         'alsa-lib' 'libpulse' 'libxcomposite' 'libxcursor' 'libxrandr'
         'libxi' 'libxinerama' 'vulkan-icd-loader')
makedepends=('git' 'mingw-w64-gcc' 'autoconf' 'bison' 'flex' 'perl' 'gettext'
             'vulkan-headers')
options=('!lto' 'staticlibs')
source=("git+https://github.com/giang17/wine.git#branch=d2d1-dcomp-11.0")
sha256sums=('SKIP')

pkgver() {
    cd "$srcdir/wine"
    printf '%s' "$(git describe --long 2>/dev/null | sed 's/^wine-//;s/\([^-]*-g\)/r\1/;s/-/./g')" \
        || printf '11.0.r0'
}

build() {
    mkdir -p "$srcdir/build"
    cd "$srcdir/build"
    "$srcdir/wine/configure" \
        --prefix=/opt/wine-d2d1 \
        --enable-archs=i386,x86_64
    make
}

package() {
    cd "$srcdir/build"
    make DESTDIR="$pkgdir" install
}
