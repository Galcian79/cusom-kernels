#
# Based on the linux package by:
# Maintainer: Jan Alexander Steffens (heftig) <heftig@archlinux.org>
#

pkgbase=linux-custom-test
_tag=v6.12.36
pkgver=6.12.36
pkgrel=1
pkgdesc="Custom Linux LTS kernel (test)"
arch=(x86_64)
url="https://kernel.org/"
license=(GPL-2.0-only)

makedepends=( bc cpio gettext git libelf pahole perl python tar xz )
options=( !debug !strip )

_srcname=linux
source=(
  "$_srcname::git+https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git#tag=$_tag"
  config
  0006.patch
  0007-futex.patch
  0007-nt.patch
  0012.patch
  0014.patch
)
validpgpkeys=(
  ABAF11C65A2970B130ABE3C479BE3E4300411886
  647F28654894E3BD457199BE38DBBDC86092693E
  83BC8889351B5DEBBB68416EB8AC08600F108CDF
)
sha256sums=(
  'SKIP' # git source
  '50cafbdd5b1aa8b3497034fe351f02e1891d0a33274eadc8afd0c784ccc5a2a3'
  '122139befc3da25b00b5119e9a2fc59b266ed0de53f36912e9e60f3287c479d6'
  '9df628fd530950e37d31da854cb314d536f33c83935adf5c47e71266a55f7004'
  '29a48e055f35358113d245cc370b85bc8bb2d697f7474e2536e137663038a1a9'
  '891874073250bd7a9e0dd2d30a54760e3c49275e47c99becebf75aa4e5c9f086'
  '899f45d080848a83aebd9a710e6fdad2c3731eab0fc455af9ac29c7b0f4ceacf'
)

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER="$pkgbase"
export KBUILD_BUILD_TIMESTAMP="$(date -u +'%Y-%m-%d %H:%M:%S %z')"

prepare() {
  cd "$_srcname"

  # versioning
  echo "-$pkgrel"            > localversion.10-pkgrel
  echo "${pkgbase#linux}"    > localversion.20-pkgname

  # apply patches
  for src in "${source[@]}"; do
    f="${src##*/}"
    [[ $f = *.patch ]] || continue
    patch -Np1 < "../$f"
  done
  rm -rf .git

  # base config + defaults
  cp ../config .config
  make olddefconfig

  # prune unused modules
  make localmodconfig

  # collect loaded modules and PCI drivers
  lsmod | tail -n +2 | awk '{print $1}' > modules.list
  lspci -k | grep -A1 "Kernel driver in use" \
    | sed -n 's/.*Kernel driver in use: //p' >> modules.list
  sort -u modules.list -o modules.list

  # re-enable only those modules
  while read mod; do
    scripts/config --file .config --module "CONFIG_${mod^^}"
  done < modules.list

  # disable debug/BTF
  cat << 'EOF' >> .config
CONFIG_DEBUG_INFO=n
CONFIG_DEBUG_INFO_BTF=n
CONFIG_DEBUG_INFO_BTF_MODULES=n
EOF

  # finalize minimal defconfig
  make olddefconfig
  diff -u ../config .config || true

  # record version
  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd "$_srcname"
  make -j"$(nproc)"
  echo "Skipping tools/bpf vmlinux.h (BTF disabled)"
}

package_linux-custom-test() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=( coreutils initramfs kmod )
  optdepends=(
    'wireless-regdb: set correct wireless channels'
    'linux-firmware: firmware images for some devices'
  )

  cd "$_srcname"
  ver=$(<version)
  dest="$pkgdir/usr/lib/modules/$ver"

  install -Dm644 arch/x86/boot/bzImage "$dest/vmlinuz"
  echo "$pkgbase" | install -Dm644 /dev/stdin "$dest/pkgbase"

  ZSTD_CLEVEL=19 \
    make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  rm -f "$dest"/build
}

package_linux-custom-test-headers() {
  pkgdesc="Headers for the $pkgdesc kernel"
  depends=( pahole )

  cd "$_srcname"
  ver=$(<version)
  builddir="$pkgdir/usr/lib/modules/$ver/build"

  install -Dt "$builddir" -m644 \
    .config Makefile Module.symvers System.map localversion.* version vmlinux

  find include arch/x86/include -type f \
    -exec install -Dm644 {} "$builddir/{}" \;

  find "$builddir" -name '*.o' -delete
  strip -v $STRIP_STATIC "$builddir/vmlinux"
}

pkgname=( linux-custom-test linux-custom-test-headers )
