#
# Based on the linux package by:
# Maintainer: Jan Alexander Steffens (heftig) <heftig@archlinux.org>
#

pkgbase=linux-custom-LTS
_tag=v6.12.36
pkgver=6.12.36
pkgrel=1
pkgdesc="Linux custom LTS"
arch=(x86_64)
url="https://kernel.org/"
license=(GPL-2.0-only)

makedepends=(
  bc
  cpio
  gettext
  git
  libelf
  pahole
  perl
  python
  tar
  xz
)
options=(
  !debug
  !strip
)

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
  '17dd9695fcd3569f1ce9980df1647cc18e2c4a97b7c4ad8454541b6c1811015f'
  '50cafbdd5b1aa8b3497034fe351f02e1891d0a33274eadc8afd0c784ccc5a2a3'
  '122139befc3da25b00b5119e9a2fc59b266ed0de53f36912e9e60f3287c479d6'
  '9df628fd530950e37d31da854cb314d536f33c83935adf5c47e71266a55f7004'
  '29a48e055f35358113d245cc370b85bc8bb2d697f7474e2536e137663038a1a9'
  '891874073250bd7a9e0dd2d30a54760e3c49275e47c99becebf75aa4e5c9f086'
  '899f45d080848a83aebd9a710e6fdad2c3731eab0fc455af9ac29c7b0f4ceacf'
)

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+-d @$SOURCE_DATE_EPOCH})"

prepare() {
  cd $_srcname

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  for src in "${source[@]}"; do
    file="${src%%::*}"
    file="${file##*/}"
    [[ $file = *.patch ]] || continue
    echo "Applying patch $file..."
    patch -Np1 < "../$file"
  done

  # remove .git to avoid dirty flag
  rm -rf .git

  echo "Setting config..."
  cp ../config .config

  # disable all debug info and BTF (core + modules)
  cat << 'EOF' >> .config
CONFIG_DEBUG_INFO=n
CONFIG_DEBUG_INFO_BTF=n
CONFIG_DEBUG_INFO_BTF_MODULES=n
EOF

  make olddefconfig
  diff -u ../config .config || :

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
  make -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils initramfs kmod)
  optdepends=(
    'wireless-regdb: to set the correct wireless channels'
    'linux-firmware: firmware images for some devices'
  )
  provides=(KSMBD-MODULE VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)

  cd $_srcname
  local ver=$(<version)
  local modulesdir="$pkgdir/usr/lib/modules/$ver"

  echo "Installing boot image..."
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 \
    make INSTALL_MOD_STRIPSUBDIR=1 \
      INSTALL_MOD_PATH="$pkgdir/usr" \
      modules_install
  rm -f "$modulesdir"/build
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local ver=$(<version)
  local builddir="$pkgdir/usr/lib/modules/$ver/build"

  install -Dt "$builddir" -m644 .config Makefile Module.symvers \
    System.map localversion.* version vmlinux \
    tools/bpf/bpftool/vmlinux.h
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  cp -a scripts "$builddir/scripts"
  ln -sr "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool
  install -Dt "$builddir/tools/bpf/resolve_btfids" \
    tools/bpf/resolve_btfids/resolve_btfids

  find include -type f -exec install -Dm644 {} "$builddir/{}" \;
  find arch/x86/include -type f -exec install -Dm644 {} "$builddir/arch/x86/{}" \;

  # cleanup and strip
  find "$builddir" -name '*.o' -delete
  strip -v $STRIP_STATIC "$builddir/vmlinux"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() { _package${_p#$pkgbase}; }"
done
