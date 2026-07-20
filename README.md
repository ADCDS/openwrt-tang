# openwrt-tang

OpenWrt packages for [**tang**](https://github.com/latchset/tang) and its
dependency [**jose**](https://github.com/latchset/jose) — neither of which is
in the official OpenWrt feeds.

Tang is the server side of **Network-Bound Disk Encryption (NBDE)**. Paired with
[clevis](https://github.com/latchset/clevis) on a client, it lets a LUKS-encrypted
machine **auto-unlock while it can reach the Tang server on the local network**,
and stay encrypted otherwise. Running Tang on an always-on router is a natural fit:
a headless box can then boot unattended (e.g. via **Wake-on-LAN**) at home, yet
remains locked if the disk or machine is taken off the network.

## What's here

| Package | Builds | Notes |
|---|---|---|
| `jose` / `libjose` | JOSE CLI + shared library | deps: `jansson`, `zlib`, `libopenssl` |
| `tang` | `tangd` server + procd service | deps: `jose`, `libhttp-parser`, `socat`, `jq` |

OpenWrt has no systemd, so instead of Tang's `tangd.socket`/`tangd@.service`
socket activation, the `tang` package ships a **procd** service (`/etc/init.d/tang`)
that uses **socat** to fork `tangd` per connection (inetd-style), configured via
UCI (`/etc/config/tang`).

## Building

Tested against **OpenWrt 25.12.5**, target `mediatek/filogic`
(`aarch64_cortex-a53`), which uses the **apk** package manager.

```sh
# 1. Get the matching SDK for your target/version
wget https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/\
openwrt-sdk-25.12.5-mediatek-filogic_gcc-14.3.0_musl.Linux-x86_64.tar.zst
tar --use-compress-program=unzstd -xf openwrt-sdk-*.tar.zst
cd openwrt-sdk-*/

# 2. Add this repo as a package feed (adjust the path)
cat >> feeds.conf.default <<'EOF'
src-link mypkgs /path/to/openwrt-tang
EOF
./scripts/feeds update base packages mypkgs
./scripts/feeds install -p mypkgs jose tang

# 3. Build
make defconfig
make package/jose/compile
make package/tang/compile

# .apk files land in bin/packages/aarch64_cortex-a53/mypkgs/
```

## Installing on the router

```sh
# copy the .apk files over, then:
apk add ./jose-*.apk ./libjose-*.apk ./tang-*.apk
```

The `tang` postinst creates an unprivileged `tang` user, a key directory at
`/var/db/tang`, and generates an initial keypair. Then:

```sh
uci set tang.tang.port='7500'      # NOT 80 (LuCI/uhttpd uses that)
uci commit tang
/etc/init.d/tang enable
/etc/init.d/tang start

# open the port on the LAN only (example)
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Tang-LAN'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest_port='7500'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall && /etc/init.d/firewall reload
```

Verify the advertisement:

```sh
curl http://<router-ip>:7500/adv    # returns a signed JWK set
```

## Binding a client (clevis + LUKS)

On the encrypted client (needs `clevis`, `clevis-luks`, `clevis-initramfs`):

```sh
clevis luks bind -d /dev/<luks-partition> tang '{"url":"http://<router-ip>:7500"}'
update-initramfs -u            # Debian/Ubuntu; regenerate so it unlocks at boot
```

The passphrase keyslot stays as a fallback. For extra resilience you can bind to
multiple pins with Shamir Secret Sharing (e.g. "unlock if Tang **or** the TPM is
available"):

```sh
clevis luks bind -d /dev/<luks-partition> sss '{"t":1,"pins":{"tang":[{"url":"..."}],"tpm2":{}}}'
```

## Security model

Auto-unlock trades some at-rest protection for unattended boot. With Tang you get
a **network boundary**: the disk unlocks only where the Tang server is reachable
(your LAN). It does **not** protect against an attacker who keeps the machine on
your network, or who controls both the client and the Tang host. Combine with a
TPM pin (SSS) if you also want hardware binding.

## License

Packaging (Makefiles, service scripts) is GPL-2.0. The upstream projects retain
their own licenses (jose: Apache-2.0, tang: GPL-3.0-or-later).
