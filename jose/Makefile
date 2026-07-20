#
# OpenWrt package: jose (JOSE - JSON Object Signing and Encryption)
# C-language implementation + CLI. Dependency of tang.
#
include $(TOPDIR)/rules.mk

PKG_NAME:=jose
PKG_VERSION:=15
PKG_RELEASE:=1

PKG_SOURCE:=jose-$(PKG_VERSION).tar.xz
PKG_SOURCE_URL:=https://github.com/latchset/jose/releases/download/v$(PKG_VERSION)
PKG_HASH:=1d055c445392aa48d709ecd6e56220384ae2b480496e270818bddf1f219c8659

PKG_MAINTAINER:=Adriel <adriel@adriel.eu>
PKG_LICENSE:=Apache-2.0
PKG_LICENSE_FILES:=COPYING

PKG_INSTALL:=1
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/meson.mk

define Package/libjose
  SECTION:=libs
  CATEGORY:=Libraries
  TITLE:=JOSE (JSON Object Signing and Encryption) shared library
  URL:=https://github.com/latchset/jose
  DEPENDS:=+jansson +zlib +libopenssl
  ABI_VERSION:=0
endef

define Package/jose
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=JOSE command line utility
  URL:=https://github.com/latchset/jose
  DEPENDS:=+libjose
endef

define Package/libjose/description
  Shared library for creating and managing JOSE objects (JWK, JWE, JWS),
  the cryptographic building blocks used by clevis/tang.
endef

define Package/jose/description
  Command line utility exposing the jose library (JWK/JWE/JWS operations).
endef

# no asciidoc/a2x in the SDK -> disable man pages
MESON_ARGS += -Ddocs=disabled

define Package/libjose/install
	$(INSTALL_DIR) $(1)/usr/lib
	$(CP) $(PKG_INSTALL_DIR)/usr/lib/libjose.so.* $(1)/usr/lib/
endef

define Package/jose/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/jose $(1)/usr/bin/
endef

$(eval $(call BuildPackage,libjose))
$(eval $(call BuildPackage,jose))
