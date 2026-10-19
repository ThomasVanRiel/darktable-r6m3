# Private package: darktable 5.6.2 with Canon EOS R6 Mark III support.
#
# Based on Arch's extra/darktable PKGBUILD for the same version, plus:
#   - LibRaw replaced by MStraeten's r6mIII branch (pinned commit)
#   - r6iii-modelmap.patch: darktable's own LibRaw camera table entry
#     (same change as unmerged darktable PR #21433)
#   - installed side by side under /opt/darktable-r6m3; the stock
#     darktable package keeps /usr
#   - CPU ONNX Runtime pinned as a source instead of downloaded by CMake;
#     the GPU runtime is per machine, see README.md

pkgname=darktable-r6m3
pkgver=5.6.2
pkgrel=2
pkgdesc='darktable with experimental Canon EOS R6 Mark III support (patched LibRaw)'
arch=(x86_64)
url='https://darktable.org'
_url='https://github.com/darktable-org/darktable'
license=(GPL-3.0-or-later)
depends=(colord-gtk
         exiv2
         flickcurl
         gmic
         graphicsmagick
         iso-codes
         jasper
         lensfun
         libavif
         libgphoto2
         libjpeg-turbo
         libjxl
         libsecret
         libxml2
         lua54
         openexr
         openjpeg2
         openmp
         osm-gps-map
         potrace
         pugixml
         zlib)
optdepends=('ghostscript: noise profile script'
            'gnuplot: noise profile script'
            'imagemagick: noise profile script'
            'opencl-driver: hardware accelerated image processing'
            'portmidi: game and midi controller input devices')
makedepends=(clang
             cmake
             desktop-file-utils
             git
             intltool
             libwebp
             llvm
             ninja
             portmidi
             python-jsonschema)

_prefix=/opt/darktable-r6m3
_archive="darktable-$pkgver"
_libraw_commit=aeb371cc2de99cf6527101f820b490680990aa73  # MStraeten/LibRaw r6mIII
_ort_ver=1.24.4  # what darktable 5.6.2's FindONNXRuntime.cmake would download

source=("$_url/releases/download/release-$pkgver/$_archive.tar.xz"{,.asc}
        "LibRaw::git+https://github.com/MStraeten/LibRaw.git#commit=$_libraw_commit"
        "https://github.com/microsoft/onnxruntime/releases/download/v$_ort_ver/onnxruntime-linux-x64-$_ort_ver.tgz"
        r6iii-modelmap.patch)
sha256sums=('d2da92c670401d350b576431f21e58bee40348e1f5078201a8a1b8c270585834'
            'SKIP'
            'SKIP'
            '3a211fbea252c1e66290658f1b735b772056149f28321e71c308942cdb54b747'
            'ca1392e631306674e54670a09003a16f9218b9650c80e7cadd7ecf73b9827c40')
validpgpkeys=(C4CBC150699956E2A3268EF5BB5CC8295B1779C9  # darktable releases <release@darktable.org>
              F10F9686652B0E949FCD94C318DCA123F949BD3B) # Pascal Obry <pascal@obry.net>

prepare() {
    cd "$_archive"

    rm -rf src/external/LibRaw
    cp -a "$srcdir/LibRaw" src/external/LibRaw
    grep -q 'R6 Mark III' src/external/LibRaw/src/tables/cameralist.cpp

    patch -Np1 -i "$srcdir/r6iii-modelmap.patch"

    # FindONNXRuntime.cmake checks its download dir before downloading. Not
    # src/external/onnxruntime: it patches ORT's CMake files in place, and
    # CMake 4 refuses writes into the source tree
    rm -rf "$srcdir/build"
    mkdir -p "$srcdir/build/_deps"
    cp -a "$srcdir/onnxruntime-linux-x64-$_ort_ver" "$srcdir/build/_deps/onnxruntime"
}

build() {
    local cmake_flags=(
        PROJECT_VERSION="$pkgver"
        CMAKE_INSTALL_PREFIX="$_prefix"
        CMAKE_BUILD_TYPE=Release
        BINARY_PACKAGE_BUILD=1
        ONNXRUNTIME_OFFLINE=On
        USE_AI=On
        USE_COLORD=On
        USE_HEIF=Off
        USE_LIBSECRET=On
        USE_LUA=On
        BUILD_CURVE_TOOLS=On
        BUILD_NOISE_TOOLS=On
        BUILD_USERMANUAL=Off
        RAWSPEED_ENABLE_LTO=On
    )
    cmake -B build -S "$_archive" -G Ninja ${cmake_flags[@]/#/-D }
    cmake --build build
}

package() {
    DESTDIR="$pkgdir" cmake --install build

    # App launchers don't look in /opt. Put a copy of the desktop entry where
    # they do, with its own name and an absolute icon path, so it can sit next
    # to the stock darktable entry.
    sed -e '/^Name\[/d' \
        -e 's|^Name=.*|Name=darktable (R6 III)|' \
        -e "s|^Icon=.*|Icon=$_prefix/share/icons/hicolor/scalable/apps/darktable.svg|" \
        "$pkgdir$_prefix/share/applications/org.darktable.darktable.desktop" |
        install -Dm644 /dev/stdin "$pkgdir/usr/share/applications/darktable-r6m3.desktop"
}
