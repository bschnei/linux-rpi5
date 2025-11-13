# Maintainer: Ben Schneider <ben@bens.haus>

pkgname=linux-rpi5
pkgver=6.12.57
_commit=ea23f08af9e5876d6076823f360693d5c32cabcc
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
  "linux-rpi.tar.gz::https://github.com/raspberrypi/linux/archive/${_commit}.tar.gz"
)
sha256sums=('45b7a94ab18df9d33e145ea9c057b3715b7262d65610bb910ee6ee8509e6f084')

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

package() {
  pkgdesc="$pkgdesc"
  depends=(
    coreutils
    kmod
  )
  optdepends=(
    'initramfs: initial ramdisk creation'
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

  for i in bcm2837 bcm2711 bcm2710; do
    rm "${modulesdir}/dtbs/broadcom/$i"*.dtb
  done

  mv "${modulesdir}/dtbs/broadcom/"* "${modulesdir}/dtbs/"
  rmdir "${modulesdir}/dtbs/broadcom"
}

# vim:set ts=8 sts=2 sw=2 et:
