# Maintainer: James Lambert (jamlam) <jamesl@mbert.onmicrosoft.com>
# Contributor: Aun-Ali Zaidi <admin@kodeit.net>
# Contributor: Jan Alexander Steffens (heftig) <jan.steffens@gmail.com>

pkgbase=mbp-16.1-linux-wifi
pkgver=5.14.6
_srcname=linux-${pkgver}
pkgrel=1
pkgdesc='Linux for MBP 16.1 Wifi'
_srctag=v${pkgver%.*}-${pkgver##*.}
url="https://git.archlinux.org/linux.git/log/?h=v$_srctag"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc kmod libelf pahole cpio perl tar xz 
  xmlto python-sphinx python-sphinx_rtd_theme graphviz imagemagick
  git
)
options=('!strip')

source=(
  https://www.kernel.org/pub/linux/kernel/v${pkgver//.*}.x/linux-${pkgver}.tar.xz
  https://www.kernel.org/pub/linux/kernel/v${pkgver//.*}.x/linux-${pkgver}.tar.sign
  config         # the main kernel config file

  # Arch Linux patches
  0001-ZEN-Add-sysctl-and-CONFIG-to-disallow-unprivileged-C.patch
  0002-HID-quirks-Add-Apple-Magic-Trackpad-2-to-hid_have_sp.patch
  
  #NVMe fix
  1001-revert-nvme.patch
  1002-reapply-tcp-patch.patch

  # Hack for AMD DC eDP link rate bug
  2001-drm-amd-display-Force-link_rate-as-LINK_RATE_RBR2-fo.patch

  # Apple SMC ACPI support
  3001-applesmc-convert-static-structures-to-drvdata.patch
  3002-applesmc-make-io-port-base-addr-dynamic.patch
  3003-applesmc-switch-to-acpi_device-from-platform.patch
  3004-applesmc-key-interface-wrappers.patch
  3005-applesmc-basic-mmio-interface-implementation.patch
  3006-applesmc-fan-support-on-T2-Macs.patch

  # T2 USB Keyboard/Touchpad support
  4001-HID-apple-Add-support-for-keyboard-backlight-on-supp.patch
  4002-HID-apple-Add-support-for-MacbookAir8-1-keyboard-tra.patch
  4003-HID-apple-Add-support-for-MacBookPro15-2-keyboard-tr.patch
  4004-HID-apple-Add-support-for-MacBookPro15-1-keyboard-tr.patch
  4005-HID-apple-Add-support-for-MacBookPro15-4-keyboard-tr.patch
  4006-HID-apple-Add-support-for-MacBookPro16-2-keyboard-tr.patch
  4007-HID-apple-Add-support-for-MacBookPro16-3-keyboard-tr.patch
  4008-HID-apple-Add-support-for-MacBookAir9-1-keyboard-tra.patch
  4009-HID-apple-Add-support-for-MacBookPro16-1-keyboard-tr.patch
  
  # MBP Peripheral support
  6001-media-uvcvideo-Add-support-for-Apple-T2-attached-iSi.patch	# UVC Camera support

  # Hack for i915 overscan issues
  7001-drm-i915-fbdev-Discard-BIOS-framebuffers-exceeding-h.patch

  # Broadcom WIFI/BT device support
  8001-corellium-wifi-bigsur.patch
  8002-brcmfmac-4377-mod.patch
  8003-brcmfmac-4377-64bit-regs.patch
  8004-brcmfmac-4377-chip-ids.patch
)

validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
)

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
  cd $_srcname

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting config..."
  cp ../config .config
  make olddefconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
  make htmldocs
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)
  provides=("linux=$pkgver")

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  provides=("linux-headers=$pkgver")

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
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
    case "$(file -bi "$file")" in
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

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers" "$pkgbase-docs")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

sha256sums=('54848c1268771ee3515e4c33e29abc3f1fa90d8144894cce6d0ebc3b158bccec'
            'SKIP'
            '1ddd4443470ad66aff8075e0528ad7757de41d474152db1362e23be72e243919'
            '6b4da532421cac5600d09c0c52742aa52d848af098f7853abe60c02e9d0a3752'
            '2184069ab00ef43d9674756e9b7a56d15188bc4494d34425f04ddc779c52acd8'
            '78ea4702928d87bfdd41c6998154d7c2e1396c823a012d3891f1b803198712aa'
            'ec658c85b9cfaa32acda11ec768e608bafe5b53a7511523b55c8c082d8dce04a'
            '786dfc22e4c6ece883e7dedd0ba3f6c14018584df95450b2cb78f3da8b01f7cb'
            '7366a08383900a09f8e742b1e4f0a02e0839a385e68e70a89d1815c197df3300'
            '8d8401a99a9dfbc41aa2dc5b6a409a19860b1b918465e19de4a4ff18de075ea3'
            '08d165106fe35b68a7b48f216566951a5db0baac19098c015bcc81c5fcba678d'
            '459906cab172df9f6712a4168e7a5d529f85b2bb58a068f2d44746df14a6d27a'
            '2827dab6eeb2d2a08034938024f902846b5813e967a0ea253dc1ea88315da383'
            '398dec7d54c6122ae2263cd5a6d52353800a1a60fd85e52427c372ea9974a625'
            '11565cff9c6a7db8846dc7d5930419045e9527863b8df5979a7465006211bd16'
            '83f4be6849ba4d5f9fad647ad2eb78bf6409ee98a40ac62e8a5b80496233d70a'
            '44bd3643b2b22fedc59d79511199f30ce6759fa0acdd9a66262a53c5e046da6b'
            'eb04a492197783643b3e72b1d0cf0e856290381997bd165a14fbc63ac1489c25'
            '69d56d42473526f7dbd4fb18c5e1baafe4e6d32995b2506bd48ff981c53b5385'
            '1deeacae1875cf9075b858a8bfb2463ebc531c9030b7c2ab46bbb8e4c3b974db'
            '40eff5e88bb30c51c6b97e85c2e7b8dec5f97916f768e6c07618d9c5afe68574'
            'cac035fe07663a319185c644c5b39b34bef89ada348881fa4a02d15290260445'
            '9dfa9f02d17c5cd9620fa2c1d43ca967b81b6a56d33c2bafae14e0c64e498baa'
            '9640178d6251686c980c30fc528b3d70beac6ce8246bf433506a3f843808326c'
            '90a6012cdd8a64ede8e0bbaf7331960bd68f628e0973b65459188eb1ccb5b829'
            '903c9e2d141ddb4ebc7f60fd08b54d97306a187a06bfc8832bc8f442f00027e3'
            'e82c6bc63ef94745a576b7433f691c17ab2065b99b1b944e25ee005b0b662dec'
            'a57c31b207fcb62fc2b764b3e69058cf6f0291a34724105252d65e7fa06b1905'
            'c8442b069604d2f6ad566e64eb6ebbafbd44e04799ff625419b3552154d713c1')
