#
# OpenWrt package: tang (network-bound encryption server)
# Serves LUKS key recovery for clevis (McCallum-Relyea). No systemd on
# OpenWrt, so tangd is socket-activated via socat from a procd service.
#
include $(TOPDIR)/rules.mk

PKG_NAME:=tang
PKG_VERSION:=15
PKG_RELEASE:=1

PKG_SOURCE:=tang-$(PKG_VERSION).tar.xz
PKG_SOURCE_URL:=https://github.com/latchset/tang/releases/download/v$(PKG_VERSION)
PKG_HASH:=eaf4a2abfea3d05f454a8841e98332be1e1e2432744c70bb7765651ed82c3f7c

PKG_MAINTAINER:=Adriel <adriel@adriel.eu>
PKG_LICENSE:=GPL-3.0-or-later
PKG_LICENSE_FILES:=COPYING

PKG_INSTALL:=1
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/meson.mk

define Package/tang
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=Services
  TITLE:=Tang network-bound encryption (NBDE) server
  URL:=https://github.com/latchset/tang
  DEPENDS:=+jose +libhttp-parser +socat +jq
endef

define Package/tang/description
  Tang implements the server side of Network-Bound Disk Encryption (NBDE).
  Combined with clevis on a client, it lets a LUKS volume auto-unlock while it
  can reach this server on the network, and stay locked otherwise. Useful for
  headless machines that need unattended boot (e.g. Wake-on-LAN).
endef

define Package/tang/conffiles
/etc/config/tang
endef

# a2x (asciidoc) is not in the SDK; the meson build treats man pages as
# best-effort, but be explicit that we don't ship them here.
MESON_ARGS += \
	-Duser=tang \
	-Dgroup=tang

define Package/tang/install
	# tangd + helper scripts land in /usr/libexec
	$(INSTALL_DIR) $(1)/usr/libexec
	$(CP) $(PKG_INSTALL_DIR)/usr/libexec/tangd $(1)/usr/libexec/
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/libexec/tangd-keygen $(1)/usr/libexec/
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/libexec/tangd-rotate-keys $(1)/usr/libexec/
	# tang-show-keys client helper (bindir)
	-$(INSTALL_DIR) $(1)/usr/bin
	-$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/tang-show-keys $(1)/usr/bin/
	# OpenWrt service integration (procd + socat), replaces systemd units
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/tang.init $(1)/etc/init.d/tang
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/tang.config $(1)/etc/config/tang
endef

define Package/tang/postinst
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] && exit 0
# create unprivileged tang user/group if absent
grep -q "^tang:" /etc/group  2>/dev/null || echo "tang:x:390:" >> /etc/group
grep -q "^tang:" /etc/passwd 2>/dev/null || echo "tang:x:390:390:tang:/var/db/tang:/bin/false" >> /etc/passwd
# key directory
mkdir -p /var/db/tang
chown tang:tang /var/db/tang
chmod 750 /var/db/tang
# generate an initial signing/derivation keypair if none exist
if [ -z "$$(ls -A /var/db/tang 2>/dev/null)" ]; then
	/usr/libexec/tangd-keygen /var/db/tang
	chown tang:tang /var/db/tang/*
fi
exit 0
endef

$(eval $(call BuildPackage,tang))
