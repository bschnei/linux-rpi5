# Maintainer: Ben Schneider <ben@bens.haus>

pkgbase=linux-rpi5
pkgver=6.12.58
_commit=1f204175932fe20e477da83012d9fd2fce221366
_srcname=linux-${_commit}
pkgrel=1
pkgdesc='Vendor kernel and modules for Raspberry Pi 5'
arch=(aarch64)
url='https://www.raspberrypi.com/'
license=('GPL-2.0 WITH Linux-syscall-note')
makedepends=(
  bc
)
options=(
  !debug
  !strip
)
install=$pkgbase.install
source=(
  "linux-rpi.tar.gz::https://github.com/raspberrypi/linux/archive/${_commit}.tar.gz"
  "config.txt"
)
sha256sums=('7f40ea4ca483d74f3f0a510c42481b18374ff312547a5bcf4642a6c78a0a47b0'
            '9e9c2c7ff18dbae6fd6c34112478bc20b49fec7bd36d08f82686f074b5bf8761')

case "${CARCH}" in
 x86_64)  KARCH=x86 ;;
 aarch64) KARCH=arm64 ;;
esac
export KARCH

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

pkgver() {
  cd "${srcdir}/${_srcname}"
  make -s kernelrelease | sed 's/-.*//'
}

prepare() {
  cd "${srcdir}/${_srcname}"

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname
  
  echo "Setting config..."

  # unset vendor LOCALVERSION to keep pkgver clean
  sed -i '/^CONFIG_LOCALVERSION=/d' ./arch/arm64/configs/bcm2712_defconfig

  # enable loading compressed firmware files which
  # is how they are packaged in linux-firmware
  echo "CONFIG_FW_LOADER_COMPRESS=y" >> ./arch/arm64/configs/bcm2712_defconfig
  echo "CONFIG_FW_LOADER_COMPRESS_ZSTD=y" >> ./arch/arm64/configs/bcm2712_defconfig
 
  # enable landlock -- consistent with Arch Linux 
  echo "CONFIG_SECURITY_LANDLOCK=y" >> ./arch/arm64/configs/bcm2712_defconfig
  echo 'CONFIG_LSM="landlock"' >> ./arch/arm64/configs/bcm2712_defconfig

  make bcm2712_defconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd "${srcdir}/${_srcname}"
  export KCFLAGS=' -mcpu=cortex-a76'
  export KCPPFLAGS=' -mcpu=cortex-a76'
  make Image.gz dtbs modules
}

_package() {
  pkgdesc="$pkgdesc"
  depends=(
    coreutils
    initramfs
    kmod
  )
  optdepends=(
    'linux-firmware-rpi5: bluetooth/wifi drivers'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(WIREGUARD-MODULE)

  cd "${srcdir}/${_srcname}"
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build link
  rm "$modulesdir"/build

  echo "Installing device tree binaries..."
  mkdir -p "${modulesdir}/dtbs"
  make INSTALL_DTBS_PATH="${modulesdir}/dtbs" dtbs_install
  install -Dt "${pkgdir}/boot" "${modulesdir}/dtbs/broadcom/bcm2712"*
  install -Dt "${pkgdir}/boot/overlays" "${modulesdir}/dtbs/overlays/"*
  rm -rf "${modulesdir}/dtbs"

  cd "${srcdir}"
  install -m644 -Dt "${modulesdir}/boot" config.txt
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/${KARCH}" -m644 "arch/${KARCH}/Makefile"
  cp -t "$builddir" -a scripts
  ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"

  # required when STACK_VALIDATION is enabled
  if grep -q "CONFIG_HAVE_STACK_VALIDATION=y" "../config.${KARCH}"; then
    install -Dt "$builddir/tools/objtool" tools/objtool/objtool
  fi

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/${KARCH}" -a "arch/${KARCH}/include"
  install -Dt "$builddir/arch/${KARCH}/kernel" -m644 "arch/${KARCH}/kernel/asm-offsets.s"

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Installing Rust files..."
  if grep -q "CONFIG_RUST=y" "../config.${KARCH}"; then
    install -Dt "$builddir/rust" -m644 rust/*.rmeta
    install -Dt "$builddir/rust" rust/*.so
  fi

  echo "Installing unstripped VDSO..."
  make INSTALL_MOD_PATH="$pkgdir/usr" vdso_install \
    link=  # Suppress build-id symlinks

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */"${KARCH}"/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    _package${_p#${pkgbase}}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
