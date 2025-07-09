pkgbase=linux-custom-test
_tag=v6.12.36
pkgver=6.12.36
pkgrel=1
pkgdesc="Custom Linux LTS kernel (DKMS friendly)"
arch=(x86_64)
url="https://kernel.org/"
license=(GPL-2.0-only)

makedepends=(bc cpio gettext git libelf pahole perl python tar xz)
options=(!debug !strip)

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
sha256sums=('SKIP'  # git source
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

  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  for src in "${source[@]}"; do
    [[ $src = *.patch ]] || continue
    patch -Np1 < "../${src##*/}"
  done
  rm -rf .git

  cp ../config .config
  make olddefconfig
  make localmodconfig

  echo "Forcing DKMS-compatible module config"
  scripts/config --file .config \
    --enable CONFIG_MODULES \
    --enable CONFIG_MODVERSIONS \
    --enable CONFIG_MODULE_UNLOAD \
    --enable CONFIG_KALLSYMS \
    --enable CONFIG_CRYPTO \
    --enable CONFIG_NET \
    --enable CONFIG_USB_SUPPORT \
    --enable CONFIG_BLK_DEV

  lsmod | tail -n +2 | awk '{print $1}' > modules.list
  lspci -k | grep -A1 "Kernel driver in use" | \
    sed -n 's/.*Kernel driver in use: //p' >> modules.list
  sort -u modules.list -o modules.list
  while read mod; do
    scripts/config --file .config --module "CONFIG_${mod^^}"
  done < modules.list

  cat << 'EOF' >> .config
CONFIG_DEBUG_INFO=n
CONFIG_DEBUG_INFO_BTF=n
CONFIG_DEBUG_INFO_BTF_MODULES=n
EOF

  make olddefconfig
  make -s kernelrelease > version
}

build() {
  cd "$_srcname"
  make -j$(nproc)
}

package_linux-custom-test() {
  pkgdesc="Kernel and modules for $pkgbase"
  depends=(coreutils initramfs kmod)
  optdepends=(
    'wireless-regdb: regulatory DB for wireless devices'
    'linux-firmware: binary firmware images'
  )

  cd "$_srcname"
  ver=$(<version)
  moddir="$pkgdir/usr/lib/modules/$ver"

  install -Dm644 arch/x86/boot/bzImage "$moddir/vmlinuz"
  echo "$pkgbase" | install -Dm644 /dev/stdin "$moddir/pkgbase"

  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install
  rm -f "$moddir/build"
}

package_linux-custom-test-headers() {
  pkgdesc="Header files for $pkgbase to build external modules"
  depends=(pahole)

  cd "$_srcname"
  ver=$(<version)
  builddir="$pkgdir/usr/lib/modules/$ver/build"

  install -Dt "$builddir" -m644 \
    .config Makefile Module.symvers System.map version vmlinux \
    localversion.* tools/bpf/bpftool/vmlinux.h 2>/dev/null || true

  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  cp -a scripts "$builddir/scripts"

  install -Dt "$builddir/tools/objtool" tools/objtool/objtool
  install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids 2>/dev/null || true

  find include arch/x86/include -type f -exec install -Dm644 {} "$builddir/{}" \;

  # symlink for DKMS
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"

  find "$builddir" -name '*.o' -delete
  strip -v $STRIP_STATIC "$builddir/vmlinux"
}

pkgname=(linux-custom-test linux-custom-test-headers)
