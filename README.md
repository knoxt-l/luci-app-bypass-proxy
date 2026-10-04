# Bypass Proxy for OpenWrt

Selective **TCP-only** SOCKS proxy for OpenWrt with a LuCI page.
Only connections from your LAN to the IPv4 addresses / CIDR networks you list are sent through a
SOCKS5 or SOCKS4 server. Everything else uses the normal WAN connection.

```
LAN client
    |
    v
iptables-nft  nat PREROUTING  (-i br-lan, -p tcp)  ->  chain BYPASS_PROXY
    |
    +-- destination in your list --> REDIRECT :1337 --> Redsocks --> SOCKS4/SOCKS5 server
    |
    +-- everything else ----------------------------> normal WAN
```

* TCP only, IPv4 only. No UDP, no HTTP/HTTPS proxies.
* Router-originated and WAN traffic is never redirected.
* One `REDIRECT` rule per destination; startup fails loudly if any rule cannot be installed.
* Loads `nf_nat`, `nft_compat`, `iptable_nat`, `xt_REDIRECT` and retries until `REDIRECT` really works (boot-safe).
* SOCKS5 username/password: two independent optional fields, escaped safely in the Redsocks config (mode `600`).
* Nothing is preconfigured: **no default proxy server, no default destinations, no credentials.**

Built for OpenWrt 25.12 (`apk`, `iptables-nft`). Other versions are untested.

## Install

On the router (SSH, as root). Replace `<USER>/<REPO>` with this repository:

```sh
wget -O /tmp/bypass-proxy-install.sh https://raw.githubusercontent.com/<USER>/<REPO>/main/install.sh
sh /tmp/bypass-proxy-install.sh
```

The installer installs missing dependencies (`redsocks`, `iptables-nft`, `kmod-ipt-nat`, `luci-compat`),
writes the files, enables start at boot, and keeps any existing `/etc/config/bypass_proxy`.
It does not touch the rest of your router configuration. Because nothing is configured yet, it will not start the service.

## Configure

LuCI: **Services -> Bypass Proxy** - set *Proxy Server*, *Proxy Port*, *Proxy Type*, optional
*Username* / *Password*, add at least one *Proxy Destination*, **Save & Apply**, then **Start**.
(The button shows *Stop* while running; *Restart* is separate; *Start at boot* toggles boot start.)

Or with `uci` (documentation addresses used as examples):

```sh
uci set bypass_proxy.main.server='proxy.example.com'
uci set bypass_proxy.main.port='1080'
uci set bypass_proxy.main.type='socks5'          # or socks4
uci set bypass_proxy.main.username='myuser'      # optional
uci set bypass_proxy.main.password='mypassword'  # optional
uci add_list bypass_proxy.main.destination='203.0.113.5'
uci add_list bypass_proxy.main.destination='198.51.100.0/24'
uci commit bypass_proxy
/etc/init.d/bypass-proxy restart
```

Service control: `/etc/init.d/bypass-proxy start|stop|restart|status|enable|disable`

## Verify

```sh
sh verify.sh          # read-only checks of the running setup
sh verify.sh --full   # also tests the username/password combinations and stop/start
                      # (restarts the service a few times, restores your config)
```

From a LAN device, connect to one of your destinations, then on the router run
`iptables -t nat -L BYPASS_PROXY -n -v -x`: the matching rule's packet counter should increase.
Traffic to hosts that are not listed must not change any counter.

## Notes and limitations

* **Credentials are stored in clear text** in `/etc/config/bypass_proxy` and in the generated
  `/etc/redsocks-bypass.conf` (mode `600`). Never commit your router's config to a public repository.
* **SOCKS4 has no password authentication**; the password is ignored for SOCKS4.
* For SOCKS5 authentication set **both** username and password. Redsocks may skip authentication
  when only one of them is present.
* `/0` destinations are rejected on purpose, so the whole LAN can never be redirected by mistake.
* The LAN interface is `br-lan` (variable `LAN_IF` at the top of the init script).
* The LuCI page is a classic Lua/CBI page and needs the `luci-compat` package.
* `Start` fails with a log message if the server or destinations are missing or invalid:
  `logread -e bypass-proxy`.

## Uninstall

```sh
/etc/init.d/bypass-proxy stop
/etc/init.d/bypass-proxy disable
rm -f /etc/init.d/bypass-proxy /etc/config/bypass_proxy /etc/redsocks-bypass.conf \
      /usr/lib/lua/luci/controller/bypass_proxy.lua /usr/lib/lua/luci/model/cbi/bypass_proxy.lua
rm -rf /tmp/luci-*
```

## Development

Edit the files under `src/`, then run `sh build.sh` to regenerate the self-contained `install.sh`.

## License

Add a license of your choice (for example MIT) before publishing.
