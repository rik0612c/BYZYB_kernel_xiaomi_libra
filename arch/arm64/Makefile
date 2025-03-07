#
# arch/arm64/Makefile
#
# This file is included by the global makefile so that you can add your own
# architecture-specific flags and dependencies.
#
# This file is subject to the terms and conditions of the GNU General Public
# License.  See the file "COPYING" in the main directory of this archive
# for more details.
#
# Copyright (C) 1995-2001 by Russell King

LDFLAGS_vmlinux	:= --no-undefined -X
CPPFLAGS_vmlinux.lds = -DTEXT_OFFSET=$(TEXT_OFFSET)
OBJCOPYFLAGS	:=-O binary -R .note -R .note.gnu.build-id -R .comment -S
GZFLAGS		:=-9

KBUILD_DEFCONFIG := defconfig
KBUILD_CFLAGS	+= -fno-pic

ifeq ($(CONFIG_COMPAT_VDSO), y)
  CROSS_COMPILE_ARM32 ?= $(CONFIG_CROSS_COMPILE_ARM32:"%"=%)

  # Check that the user has provided a valid prefix for the 32-bit toolchain.
  # To prevent selecting the system CC by default, the prefix is not
  # allowed to be empty, unlike CROSS_COMPILE. In the unlikely event that the
  # system CC is actually the 32-bit ARM compiler to be used, the
  # variable can be set to the dirname (e.g. CROSS_COMPILE_ARM32=/usr/bin/).
  # Note: this Makefile is read both before and after regenerating the config
  # (if needed). Any warning appearing before the config has been regenerated
  # should be ignored. If the error is triggered and you set
  # CONFIG_CROSS_COMPILE_ARM32, set CROSS_COMPILE_ARM32 to an appropriate value
  # when invoking make and fix CONFIG_CROSS_COMPILE_ARM32.
  ifeq ($(CROSS_COMPILE_ARM32),)
    $(error CROSS_COMPILE_ARM32 not defined or empty, the compat vDSO will not be built)
  else ifeq ($(cc-name),clang)
    export CLANG_TRIPLE_ARM32 ?= $(CROSS_COMPILE_ARM32)
    export CLANG_TARGET_ARM32 := --target=$(notdir $(CLANG_TRIPLE_ARM32:%-=%))
    export GCC_TOOLCHAIN32_DIR := $(dir $(shell which $(CROSS_COMPILE_ARM32)ld))
    export GCC_TOOLCHAIN32 := $(realpath $(GCC_TOOLCHAIN32_DIR)/..)
    export CLANG_PREFIX32 := --prefix=$(GCC_TOOLCHAIN32_DIR)$(notdir $(CROSS_COMPILE_ARM32))
    export CLANG_GCC32_TC := --gcc-toolchain=$(GCC_TOOLCHAIN32)
    export CONFIG_VDSO32 := y
    vdso32 := -DCONFIG_VDSO32=1
  else ifeq ($(shell which $(CROSS_COMPILE_ARM32)gcc 2> /dev/null),)
    $(error $(CROSS_COMPILE_ARM32)gcc not found, check CROSS_COMPILE_ARM32)
  else
    export CROSS_COMPILE_ARM32
    export CONFIG_VDSO32 := y
    vdso32 := -DCONFIG_VDSO32=1
  endif
endif

KBUILD_CFLAGS	+= $(vdso32)
KBUILD_CFLAGS	+= -mgeneral-regs-only
KBUILD_CFLAGS	+= $(call cc-option, -mpc-relative-literal-loads)
KBUILD_CFLAGS	+= -fno-asynchronous-unwind-tables -fno-pic
KBUILD_AFLAGS	+= $(vdso32)

KBUILD_CFLAGS	+= $(call cc-option,-mabi=lp64)
KBUILD_AFLAGS	+= $(call cc-option,-mabi=lp64)

ifeq ($(CONFIG_CPU_BIG_ENDIAN), y)
KBUILD_CPPFLAGS	+= -mbig-endian
AS		+= -EB
LD		+= -EB
ifeq ($(ld-name),gold)
LDFLAGS		+= -maarch64_elf64_be_vec
else
LDFLAGS		+= -maarch64linuxb
endif
else
KBUILD_CPPFLAGS	+= -mlittle-endian
AS		+= -EL
LD		+= -EL
ifeq ($(ld-name),gold)
LDFLAGS		+= -maarch64_elf64_le_vec
else
LDFLAGS		+= -maarch64linux
endif
endif

CHECKFLAGS	+= -D__aarch64__

# Default value
head-y		:= arch/arm64/kernel/head.o

# The byte offset of the kernel image in RAM from the start of RAM.
TEXT_OFFSET := 0x00080000

export	TEXT_OFFSET GZFLAGS

core-y		+= arch/arm64/kernel/ arch/arm64/mm/
core-$(CONFIG_XEN)		+= arch/arm64/xen/
core-$(CONFIG_KVM) += arch/arm64/kvm/
core-$(CONFIG_CRYPTO) += arch/arm64/crypto/
libs-y		:= arch/arm64/lib/ $(libs-y)

# Default target when executing plain make
ifeq ($(CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE),y)
KBUILD_IMAGE	:= Image-dtb
ifeq ($(CONFIG_RD_GZIP),y)
KBUILD_IMAGE	:= Image.gz-dtb
endif
else
KBUILD_IMAGE	:= Image
ifeq ($(CONFIG_RD_GZIP),y)
KBUILD_IMAGE	:= Image.gz
endif
endif

KBUILD_DTBS	:= dtbs

all:	$(KBUILD_IMAGE) $(KBUILD_DTBS)

boot := arch/arm64/boot

Image%: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

zinstall install:
	$(Q)$(MAKE) $(build)=$(boot) $@

%.dtb: scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts $(boot)/dts/$@

dtbs: scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts dtbs
	$(Q)$(MAKE) $(build)=$(boot)/dts/qcom dtbs

PHONY += vdso_install
vdso_install:
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso $@

Image%-dtb: vmlinux scripts dtbs
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

# We use MRPROPER_FILES and CLEAN_FILES now
archclean:
	$(Q)$(MAKE) $(clean)=$(boot)

# We need to generate vdso-offsets.h before compiling certain files in kernel/.
# In order to do that, we should use the archprepare target, but we can't since
# asm-offsets.h is included in some files used to generate vdso-offsets.h, and
# asm-offsets.h is built in prepare0, for which archprepare is a dependency.
# Therefore we need to generate the header after prepare0 has been made, hence
# this hack.
prepare: vdso_prepare
vdso_prepare: prepare0
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso include/generated/vdso-offsets.h
	$(if $(CONFIG_VDSO32),$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso32 \
					  include/generated/vdso32-offsets.h)

define archhelp
  echo  '* Image.gz      - Compressed kernel image (arch/$(ARCH)/boot/Image.gz)'
  echo  '  Image         - Uncompressed kernel image (arch/$(ARCH)/boot/Image)'
  echo  '* dtbs          - Build device tree blobs for enabled boards'
  echo  '  install       - Install uncompressed kernel'
  echo  '  zinstall      - Install compressed kernel'
  echo  '                  Install using (your) ~/bin/installkernel or'
  echo  '                  (distribution) /sbin/installkernel or'
  echo  '                  install to $$(INSTALL_PATH) and run lilo'
endef
