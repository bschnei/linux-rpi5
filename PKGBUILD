# Maintainer: Ben Schneider <ben@bens.haus>

pkgbase=linux-rpi5
pkgver=6.18.6
_commit=e3e63c07292c97904402c830e2a8f5d98f469979
_bluezcommit=cdf61dc691a49ff01a124752bd04194907f0f9cd
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
source=(
  "${_srcname}.tar.gz::https://github.com/raspberrypi/linux/archive/${_commit}.tar.gz"
  "BCM4345C0.hcd::https://raw.githubusercontent.com/RPi-Distro/bluez-firmware/$_bluezcommit/debian/firmware/broadcom/BCM4345C0.hcd"
  "config.txt"
)
sha256sums=('03c3831d370b7aca2b6fc753f876081583072b2c3cc9d97066b9e0d7c5501ad1'
            '51c45e77ddad91a19e96dc8fb75295b2087c279940df2634b23baf71b6dea42c'
            '7672f8dcf1e326420f38a44a3116dd66b5e149d5124bc37e3a91db7cea7276f6')

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
    kmod
    mkinitcpio
  )
  optdepends=(
    'linux-firmware-rpi5: wifi and bluetooth drivers'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(WIREGUARD-MODULE)
  replaces=('linux-rpi-16k')
  install=$pkgbase.install

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
  replaces=('linux-rpi-16k-headers')

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/${KARCH}" -m644 "arch/${KARCH}/Makefile"
  cp -t "$builddir" -a scripts
  ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"

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

package_linux-firmware-rpi5() {
  depends=('linux-firmware-broadcom')
  replaces=('firmware-raspberrypi')

  mkdir -p $pkgdir/usr/lib/firmware/brcm

  # hopefully the need for this symlink goes
  # away with mainline device support. Without
  # it, the kernel fails to load the firmware for wifi.
  ln -s ../cypress/cyfmac43455-sdio.bin.zst "$pkgdir/usr/lib/firmware/brcm/brcmfmac43455-sdio.raspberrypi,5-model-b.bin.zst"

  # bluetooth
  cd "${srcdir}"
  install -m 0644 *.hcd "${pkgdir}/usr/lib/firmware/brcm"
}

pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    _package${_p#${pkgbase}}
  }"
done
pkgname+=("linux-firmware-rpi5")

# vim:set ts=8 sts=2 sw=2 et:
