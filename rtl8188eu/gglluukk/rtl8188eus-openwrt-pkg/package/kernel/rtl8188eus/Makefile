#
# Realtek RTL8188EUS/EU out-of-tree vendor driver, gglluukk/rtl8188eus fork.
#
# This package replaces the earlier SimplyCEO/rtl8188eus package.  Two things
# are load-bearing and must not be changed back:
#
#   1) It is compiled against the headers exported by package/kernel/mac80211
#      (backports), NOT against the kernel tree headers.  The cfg80211.ko that
#      ships in the image is built by the mac80211 package from backports, so
#      its struct layouts and cfg80211_ops prototypes are the backports ones.
#
#   2) KBUILD_EXTRA_SYMBOLS points at $(PKG_SYMVERS_DIR)/*.symvers.  OpenWrt's
#      include/kernel.mk collects the Module.symvers of every kernel package
#      into that directory (collect_module_symvers, PKG_SYMVERS_DIR =
#      $(KERNEL_BUILD_DIR)/symvers).  cfg80211's exported symbols live in
#      mac80211.symvers there; the kernel tree's own Module.symvers does NOT
#      contain them.  Without this, modpost reports every cfg80211_* reference
#      as "undefined!".
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=rtl8188eus
PKG_VERSION:=5.3.9.20260818
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/gglluukk/rtl8188eus.git
PKG_SOURCE_VERSION:=f48878a7a66b1281e6eab0cf2b55e7149510daf2
PKG_SOURCE_DATE:=2026-08-18
PKG_MIRROR_HASH:=skip

PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk

# The two parallel header trees exported by the mac80211 package.
# Order is fixed and must not be swapped.
RTL8188EUS_BPINC:=-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211-backport \
	-I$(STAGING_DIR)/usr/include/mac80211/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211

# NOTE: recursive '=' on purpose.  A ':=' here would be expanded while this
# Makefile is parsed, which is before the mac80211 package has been built, so
# it would silently be empty.  With '=' the wildcard is expanded when the
# recipe runs, by which time mac80211.symvers is on disk.
RTL8188EUS_SYMVERS = $(wildcard $(PKG_SYMVERS_DIR)/*.symvers)

define KernelPackage/rtl8188eus
  SUBMENU:=Wireless Drivers
  TITLE:=Realtek RTL8188EUS/EU vendor driver (backports headers)
  DEPENDS:=+kmod-cfg80211 +kmod-usb-core +@USB_SUPPORT
  FILES:=$(PKG_BUILD_DIR)/8188eu.ko
  AUTOLOAD:=$(call AutoProbe,8188eu)
endef

define KernelPackage/rtl8188eus/description
  Out-of-tree vendor driver for RTL8188EU/EUS/ETV USB adapters.
  Compiled against the backports headers exported by the mac80211 package so
  that struct ieee80211_channel and struct cfg80211_ops seen at compile time
  match the cfg80211.ko that is actually running in the image.
endef

# Do not configure this package before the wireless headers are on disk.
STAMP_CONFIGURED_DEPENDS:=$(STAGING_DIR)/usr/include/mac80211-backport/backport/autoconf.h

define Build/Prepare
	$(call Build/Prepare/Default)
	# --- 1) move the version guard down to 6.18 --------------------------
	# os_dep/linux/ioctl_cfg80211.c carries 20 x ">= KERNEL_VERSION(7,1,0)"
	# guards that switch cfg80211 callbacks from "struct net_device *" to
	# "struct wireless_dev *".  The driver must take the NEW branch because
	# the cfg80211.ko in the image comes from backports 7.2 (see
	# package/kernel/mac80211: PKG_SOURCE_VERSION:=7.2), where
	# cfg80211_new_sta() and the cfg80211_ops callbacks all take
	# "struct wireless_dev *".  The kernel is 6.18.54, so move the threshold
	# down to force the new branch.  core/rtw_br_ext.c has one more guard of
	# the same shape, but it is about <linux/if_pppox.h> and the __KERNEL__
	# macro, not cfg80211 - leave it alone.
	$(SED) 's/KERNEL_VERSION(7, 1, 0)/KERNEL_VERSION(6, 18, 0)/g' \
		$(PKG_BUILD_DIR)/os_dep/linux/ioctl_cfg80211.c
	@if grep -q 'KERNEL_VERSION(7, 1, 0)' $(PKG_BUILD_DIR)/os_dep/linux/ioctl_cfg80211.c; then \
		echo "ERROR: version guard patch did not apply - aborting"; exit 1; \
	fi
	# Do NOT touch the ">= KERNEL_VERSION(7,2,0)" guards: those select
	# strncpy-vs-strscpy and the pppoe/ddpe headers, i.e. plain kernel API,
	# and for a 6.18 kernel the 'false' branch is the correct one.
	# --- 2) keep the driver's own debug output ---------------------------
	$(SED) 's/^CONFIG_RTW_DEBUG = n/CONFIG_RTW_DEBUG = y/' $(PKG_BUILD_DIR)/Makefile
	# --- 3) shadow mac80211's sta_info.h only ----------------------------
	# Build/InstallDev of the mac80211 package copies net/mac80211/*.h into
	# .../usr/include/mac80211/, so a second sta_info.h (mac80211's, without
	# the vendor "cmn" member) lands in the include path.  This one-file shim
	# is searched first and forwards to the driver's own header, so that
	# struct sta_info and rtw_get_stainfo() stay the vendor ones.
	mkdir -p $(PKG_BUILD_DIR)/.rtl8188eus-inc
	printf '#include "../include/sta_info.h"\n' > $(PKG_BUILD_DIR)/.rtl8188eus-inc/sta_info.h
endef

define Build/Compile
	@SYM="$(RTL8188EUS_SYMVERS)"; \
	echo "==== rtl8188eus: KBUILD_EXTRA_SYMBOLS ===="; \
	echo "  $(PKG_SYMVERS_DIR)"; \
	for f in $$SYM; do echo "    $$f"; done; \
	if [ -z "$$SYM" ]; then \
		echo "ERROR: no *.symvers collected in $(PKG_SYMVERS_DIR)"; \
		echo "       The wireless stack has not been built yet.  Run:"; \
		echo "         make package/kernel/mac80211/compile"; \
		echo "       or simply 'make' so the build system orders it for you."; \
		exit 1; \
	fi
	+export STAGING_DIR=$(STAGING_DIR); \
	$(MAKE) $(PKG_JOBS) -C "$(PKG_BUILD_DIR)" \
		ARCH="$(LINUX_KARCH)" \
		CROSS_COMPILE="$(TARGET_CROSS)" \
		KSRC="$(LINUX_DIR)" \
		KVER="$(LINUX_VERSION)" \
		CONFIG_RTL8188EU=m \
		CONFIG_USB_HCI=y \
		KCFLAGS="-Wno-error -Wno-error=int-conversion -Wno-error=incompatible-pointer-types -Wno-error=implicit-function-declaration" \
		KBUILD_EXTRA_SYMBOLS="$(RTL8188EUS_SYMVERS)" \
		NOSTDINC_FLAGS="-I$(PKG_BUILD_DIR)/.rtl8188eus-inc $(RTL8188EUS_BPINC) -include backport/autoconf.h -include backport/backport.h" \
		modules
endef

$(eval $(call KernelPackage,rtl8188eus))
