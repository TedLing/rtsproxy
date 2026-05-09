include $(TOPDIR)/rules.mk

PKG_NAME:=luci-app-rtsproxy
PKG_VERSION:=1.0.0
PKG_RELEASE:=1

PKG_MAINTAINER:=plsy1
PKG_LICENSE:=GPL-3.0-only

LUCI_TITLE:=LuCI Support for RTSP Proxy (rtsproxy)
LUCI_DESCRIPTION:=A LuCI app for managing rtsproxy - High Performance RTSP Proxy with unified port support
LUCI_DEPENDS:=+rtsproxy
LUCI_PKGARCH:=all

include $(TOPDIR)/feeds/luci/luci.mk

# call BuildPackage - OpenWrt buildroot signature
