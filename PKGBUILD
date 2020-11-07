## some experiments
pkgbase=linux-custom 
pkgver=5.9.6
_srcname=linux-${pkgver}
pkgrel=3
arch=('x86_64')
url="http://www.kernel.org/"
license=('GPL2')
makedepends=('xmlto' 'docbook-xsl' 'kmod' 'inetutils' 'bc' 'libelf' 'pahole')
options=('!strip')
source=("https://www.kernel.org/pub/linux/kernel/v5.x/${_srcname}.tar.xz"
#        "https://www.kernel.org/pub/linux/kernel/v5.x/${_srcname}.tar.sign"
        'https://raw.githubusercontent.com/quarkscript/custom-linux-kernel/master/conf_tmpl'
#        'https://raw.githubusercontent.com/quarkscript/custom-linux-kernel/master/cpu.patch'
#        'https://raw.githubusercontent.com/quarkscript/custom-linux-kernel/master/cpu_5.8.1.patch'
        'https://raw.githubusercontent.com/quarkscript/custom-linux-kernel/master/cpu_5.9.6.patch'
        'https://raw.githubusercontent.com/quarkscript/Simple_func_scripts/master/sfslib'
        )
sha512sums=('SKIP' 'SKIP' 'SKIP' 'SKIP')
validpgpkeys=(
              'ABAF11C65A2970B130ABE3C479BE3E4300411886' # Linus Torvalds
              '647F28654894E3BD457199BE38DBBDC86092693E' # Greg Kroah-Hartman
             )
_kernelname=${pkgbase#linux}

MAKEFLAGS+=" -j$(($(grep 'model name' /proc/cpuinfo --count)+1))"

prepare() {
  # try CPU-optimization patch
  cp sfslib "${srcdir}/${_srcname}/sfslib"
  #cp cpu.patch "${srcdir}/${_srcname}/cpu.patch"
  #cp cpu_5.8.1.patch "${srcdir}/${_srcname}/cpu.patch"
  cp cpu_5.9.6.patch "${srcdir}/${_srcname}/cpu.patch"
  cp conf_tmpl "${srcdir}/${_srcname}/conf_tmpl"
  cd "${srcdir}/${_srcname}"
  chmod +x sfslib
  
  temp_var=$(./sfslib localcpu)
  
  patch -p1 -i localcpu.patch
  
  yes '' | make ${MAKEFLAGS} localmodconfig
  
  echo '
  force integrate template to generated kernel config
  '
  ./sfslib fti
  if $(echo $temp_var | grep -q "select '"); then
    cpu_archite=$(echo $temp_var | sed "s/.*select '//g" | sed "s/'.*//g")
    echo "CONFIG_M${cpu_archite^^}=y">>.config
  else
    #echo "CONFIG_MNATIVE=y">>.config
    echo "CONFIG_GENERIC_CPU=y">>.config
  fi
  
  echo "
  additional force custom flags
  "  
  l1cs=$(cat cxxflags.txt | sed 's/.*l1-cache-size=//g' | sed 's/ .*//g')
  l2cs=$(cat cxxflags.txt | sed 's/.*l1-cache-size=//g' | sed 's/ .*//g')
  if [ "$l1cs" -gt 8 ]&&[ "$l2cs" -gt 32 ]; then
    cachesparams="--param l1-cache-size=$l1cs --param l2-cache-size=$l2cs"
  else
    cachesparams=""
  fi
  for tmpcycle in $(find -name Makefile); do
    sed -i "s/-O2/-O2 $cachesparams -faggressive-loop-optimizations -fguess-branch-probability -floop-interchange -floop-nest-optimize -floop-unroll-and-jam -fmove-loop-invariants -fomit-frame-pointer -foptimize-sibling-calls -fsplit-ivs-in-unroller -fsplit-loops -fsel-sched-pipelining -fsel-sched-pipelining-outer-loops -fpredictive-commoning -fprefetch-loop-arrays -ftree-loop-optimize -ftree-loop-distribution /g" $tmpcycle
  done
  
  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile
  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh
  make ${MAKEFLAGS} menuconfig # CLI menu for configuration
  # rewrite configuration
  yes "" | make config >/dev/null
}

build() {
  cd "${srcdir}/${_srcname}"
  export CXXFLAGS+=$(cat cxxflags.txt)
  make ${MAKEFLAGS} LOCALVERSION= bzImage modules
}

_package() {
  pkgdesc="The ${pkgbase/linux/Linux} kernel and modules"
  [ "${pkgbase}" = "linux" ] && groups=('base')
  depends=('coreutils' 'linux-firmware' 'kmod' 'mkinitcpio>=0.7')
  optdepends=('crda: to set the correct wireless channels of your country')
  backup=("etc/mkinitcpio.d/${pkgbase}.preset")
  cd "${srcdir}/${_srcname}"
  KARCH=x86
  # get kernel version
  _kernver="$(make LOCALVERSION= kernelrelease)"
  _basekernel=${_kernver%%-*}
  _basekernel=${_basekernel%.*}
  mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
  make ${MAKEFLAGS} LOCALVERSION= INSTALL_MOD_PATH="${pkgdir}" modules_install
  cp arch/$KARCH/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgbase}"

## gen. req. files
KERNEL_NAME='${KERNEL_NAME}'
KERNEL_VERSION='${KERNEL_VERSION}'
cat<<EOF>"${startdir}/${pkgbase}.pkg"
KERNEL_NAME=${_kernelname}
KERNEL_VERSION=${_kernver}

post_install () {
  # updating module dependencies
  echo ">>> Updating module dependencies. Just wait ..."
  depmod ${KERNEL_VERSION}
  echo ">>> Generating initial ramdisk, using mkinitcpio. Just wait..."
  mkinitcpio -p linux${KERNEL_NAME}
}

post_upgrade() {
  if findmnt --fstab -uno SOURCE /boot &>/dev/null && ! mountpoint -q /boot; then
    echo "WARNING: /boot appears to be a separate partition but is not mounted."
  fi

  # updating module dependencies
  echo ">>> Updating module dependencies. Just wait ..."
  depmod ${KERNEL_VERSION}
  echo ">>> Generating initial ramdisk, using mkinitcpio. Just wait..."
  mkinitcpio -p linux${KERNEL_NAME}
}

post_remove() {
  # also remove the compat symlinks
  rm -f boot/initramfs-linux${KERNEL_NAME}.img
  rm -f boot/initramfs-linux${KERNEL_NAME}-fallback.img
}
EOF
##
true && install=${pkgbase}.pkg  
##
cat<<EOF>"${srcdir}/${pkgbase}.preset"
# mkinitcpio preset file for the 'linux' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux${_kernelname}"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
default_image="/boot/initramfs-linux${_kernelname}.img"
#default_options=""

#fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/initramfs-linux${_kernelname}-fallback.img" 
fallback_options="-S autodetect"
EOF
##
cat<<EOF>"${srcdir}/60-$pkgbase.hook"
[Trigger]
Type = File
Operation = Install
Operation = Upgrade
Operation = Remove
Target = usr/lib/modules/$_kernver/*
Target = usr/lib/modules/"extramodules-${_basekernel}${_kernelname}"/*

[Action]
Description = Updating $pkgbase module dependencies...
When = PostTransaction
Exec = /usr/bin/depmod $_kernver
EOF
##
cat<<EOF>"${srcdir}/90-$pkgbase.hook"
[Trigger]
Type = File
Operation = Install
Operation = Upgrade
Target = usr/lib/modules/$_kernver/vmlinuz
Target = usr/lib/initcpio/*

[Action]
Description = Updating $pkgbase initcpios...
When = PostTransaction
Exec = /usr/bin/mkinitcpio -p $pkgbase
EOF
## end gen. req. files

  install -D -m644 "${srcdir}/${pkgbase}.preset" "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"
  #install -D -m644 "${srcdir}/60-$pkgbase.hook" "$pkgdir/usr/share/libalpm/hooks/60-$pkgbase.hook"
  #install -D -m644 "${srcdir}/90-$pkgbase.hook" "$pkgdir/usr/share/libalpm/hooks/90-$pkgbase.hook"
  # remove build and source links
  rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
  # remove the firmware
  rm -rf "${pkgdir}/lib/firmware"
  # make room for external modules
  ln -s "../extramodules-${_basekernel}${_kernelname}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
  # add real version for building modules and running depmod from post_install/upgrade
  mkdir -p "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname}"
  echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname}/version"
  # Now we call depmod...
  depmod -b "${pkgdir}" -F System.map "${_kernver}"
  # move module tree /lib -> /usr/lib
  mkdir -p "${pkgdir}/usr"
  mv "${pkgdir}/lib" "${pkgdir}/usr/"
  # add vmlinux
  install -D -m644 vmlinux "${pkgdir}/usr/lib/modules/${_kernver}/build/vmlinux" 
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for ${pkgbase/linux/Linux} kernel"
  install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"
  cd "${srcdir}/${_srcname}"
  install -D -m644 Makefile \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/Makefile"
  install -D -m644 kernel/Makefile \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/kernel/Makefile"
  install -D -m644 .config \
    "${pkgdir}/usr/lib/modules/${_kernver}/build/.config"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/include"
  for i in $(ls include/); do
    cp -a include/${i} "${pkgdir}/usr/lib/modules/${_kernver}/build/include/"
  done
  # copy arch includes for external modules
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/x86"
  cp -a arch/x86/include "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/x86/"
  # copy files necessary for later builds, like nvidia and vmware
  cp Module.symvers "${pkgdir}/usr/lib/modules/${_kernver}/build"
  cp -a scripts "${pkgdir}/usr/lib/modules/${_kernver}/build"
  # fix permissions on scripts dir
  chmod og-w -R "${pkgdir}/usr/lib/modules/${_kernver}/build/scripts"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/.tmp_versions"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/kernel"
  cp arch/${KARCH}/Makefile "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/"
  cp arch/${KARCH}/kernel/asm-offsets.s "${pkgdir}/usr/lib/modules/${_kernver}/build/arch/${KARCH}/kernel/"
  # add dm headers
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/md"
  cp drivers/md/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/drivers/md"
  # add inotify.h
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/include/linux"
  cp include/linux/inotify.h "${pkgdir}/usr/lib/modules/${_kernver}/build/include/linux/"
  # add wireless headers
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/net/mac80211/"
  cp net/mac80211/*.h "${pkgdir}/usr/lib/modules/${_kernver}/build/net/mac80211/"
  # add xfs and shmem for aufs building
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/fs/xfs"
  mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/mm"
  # copy in Kconfig files
  for i in $(find . -name "Kconfig*"); do
    mkdir -p "${pkgdir}"/usr/lib/modules/${_kernver}/build/`echo ${i} | sed 's|/Kconfig.*||'`
    cp ${i} "${pkgdir}/usr/lib/modules/${_kernver}/build/${i}"
  done
  # add objtool for external module building and enabled VALIDATION_STACK option
  if [ -f tools/objtool/objtool ];  then
      mkdir -p "${pkgdir}/usr/lib/modules/${_kernver}/build/tools/objtool"
      cp -a tools/objtool/objtool ${pkgdir}/usr/lib/modules/${_kernver}/build/tools/objtool/ 
  fi
  chown -R root.root "${pkgdir}/usr/lib/modules/${_kernver}/build"
  find "${pkgdir}/usr/lib/modules/${_kernver}/build" -type d -exec chmod 755 {} \;
  # strip scripts directory
  find "${pkgdir}/usr/lib/modules/${_kernver}/build/scripts" -type f -perm -u+w 2>/dev/null | while read binary ; do
    case "$(file -bi "${binary}")" in
      *application/x-sharedlib*) # Libraries (.so)
        /usr/bin/strip ${STRIP_SHARED} "${binary}";;
      *application/x-archive*) # Libraries (.a)
        /usr/bin/strip ${STRIP_STATIC} "${binary}";;
      *application/x-executable*) # Binaries
        /usr/bin/strip ${STRIP_BINARIES} "${binary}";;
    esac
  done
  # remove unneeded architectures
  rm -rf "${pkgdir}"/usr/lib/modules/${_kernver}/build/arch/{alpha,arc,arm,arm26,arm64,avr32,blackfin,c6x,cris,frv,h8300,hexagon,ia64,m32r,m68k,m68knommu,metag,mips,microblaze,mn10300,openrisc,parisc,powerpc,ppc,s390,score,sh,sh64,sparc,sparc64,tile,unicore32,um,v850,xtensa}
  # remove a files already in linux-docs package
  rm -f "${pkgdir}/usr/lib/modules/${_kernver}/build/Documentation/kbuild/Kconfig.recursion-issue-01"
  rm -f "${pkgdir}/usr/lib/modules/${_kernver}/build/Documentation/kbuild/Kconfig.recursion-issue-02"
  rm -f "${pkgdir}/usr/lib/modules/${_kernver}/build/Documentation/kbuild/Kconfig.select-break"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#${pkgbase}}")
    _package${_p#${pkgbase}}
  }"
done
