#
# Realtek RTL8188EUS/EU out-of-tree vendor driver.
#
# Built against the headers exported by package/kernel/mac80211 (backports),
# NOT against the kernel tree headers: the cfg80211.ko shipped in the image
# comes from backports, and its struct layouts differ from the kernel ones
# (e.g. struct ieee80211_channel is 80 bytes vs 60 bytes on armv7).
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=rtl8188eus
PKG_VERSION:=2026.10.08~491f34d1
PKG_RELEASE:=5
PKG_LICENSE:=GPL-2.0

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/SimplyCEO/rtl8188eus.git
PKG_SOURCE_VERSION:=491f34d1e7e40b6ca3ef91dc9b6563faca01d961
PKG_SOURCE_DATE:=2026-10-08
PKG_MIRROR_HASH:=skip

include $(INCLUDE_DIR)/package.mk

# Two parallel header trees exported by the mac80211 package.
# Order is fixed and must not be swapped.
RTL8188EUS_BPINC:=-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211-backport \
	-I$(STAGING_DIR)/usr/include/mac80211/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211

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
	# os_dep/linux/ioctl_cfg80211.c carries 21 x ">= KERNEL_VERSION(7,1,0)"
	# guards that switch cfg80211 callbacks from "struct net_device *" to
	# "struct wireless_dev *".  The driver must take the NEW branch because
	# the cfg80211.ko in the image comes from backports 7.2.
	# core/rtw_br_ext.c has one more guard of the same shape, but it is about
	# <linux/if_pppox.h> and the __KERNEL__ macro, not cfg80211 - leave it.
	$(SED) 's/KERNEL_VERSION(7, 1, 0)/KERNEL_VERSION(6, 18, 0)/g' \
		$(PKG_BUILD_DIR)/os_dep/linux/ioctl_cfg80211.c
	@if grep -q 'KERNEL_VERSION(7, 1, 0)' $(PKG_BUILD_DIR)/os_dep/linux/ioctl_cfg80211.c; then \
		echo "ERROR: version guard patch did not apply - aborting"; exit 1; \
	fi
	# --- 2) keep the driver's own debug output ---------------------------
	$(SED) 's/^CONFIG_RTW_DEBUG = n/CONFIG_RTW_DEBUG = y/' $(PKG_BUILD_DIR)/Makefile
	# --- 3) shadow mac80211's sta_info.h only ----------------------------
	# Build/InstallDev of the mac80211 package copies net/mac80211/*.h into
	# .../usr/include/mac80211/, so a second sta_info.h (mac80211's, without
	# the vendor "cmn" member) lands in the include path.  This one-file shim
	# is searched first and forwards to the driver's own header, so that
	# struct sta_info and rtw_get_stainfo() stay the vendor ones.  Nothing
	# else about the include order is changed.
	mkdir -p $(PKG_BUILD_DIR)/.rtl8188eus-inc
	printf '#include "../include/sta_info.h"\n' > $(PKG_BUILD_DIR)/.rtl8188eus-inc/sta_info.h
endef

define Build/Compile
	+export STAGING_DIR=$(STAGING_DIR); \
	$(MAKE) -C "$(PKG_BUILD_DIR)" \
		ARCH="$(LINUX_KARCH)" \
		CROSS_COMPILE="$(TARGET_CROSS)" \
		KSRC="$(LINUX_DIR)" \
		KVER="$(LINUX_VERSION)" \
		CONFIG_RTL8188EU=m \
		CONFIG_USB_HCI=y \
		KCFLAGS="-Wno-error -Wno-error=int-conversion -Wno-error=incompatible-pointer-types -Wno-error=implicit-function-declaration" \
		KBUILD_EXTRA_SYMBOLS="$(wildcard $(PKG_SYMVERS_DIR)/*.symvers)" \
		NOSTDINC_FLAGS="-I$(PKG_BUILD_DIR)/.rtl8188eus-inc $(RTL8188EUS_BPINC) -include backport/autoconf.h -include backport/backport.h" \
		modules
endef

$(eval $(call KernelPackage,rtl8188eus))
