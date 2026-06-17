# AdGuard Home for pfSense

This is an AdGuard Home plugin for pfSense CE/Plus, packaged as a standard FreeBSD `pkg`.

After installation, it provides:

- rc.d service: `/usr/local/etc/rc.d/adguardhome`
- pfSense menu entry: `Services > AdGuard Home`
- Web management page: `/usr/local/www/adguardhome.php`
- AdGuard Home binary: `/usr/local/bin/AdGuardHome`
- Configuration directory: `/usr/local/etc/adguardhome`
- Working directory: `/var/db/adguardhome`
- Log file: `/var/log/adguardhome.log`

## Recommended DNS Flow

pfSense uses Unbound DNS Resolver on port `53` by default. To make AdGuard Home effective for LAN clients, the recommended flow is:

```text
Client -> AdGuard Home:53 -> Unbound DNS Resolver:5353 -> Upstream DNS
```

This means:

- LAN clients continue to use pfSense port `53` as their DNS server.
- AdGuard Home listens on `0.0.0.0:53` and handles ad blocking, query logs, and statistics.
- Unbound is moved to `127.0.0.1:5353` and continues to provide pfSense DNS Resolver behavior.
- AdGuard Home uses `127.0.0.1:5353` as its upstream DNS server.

The package does not automatically change the Unbound port, so existing DNS behavior is not changed during installation. Switch the DNS flow manually after confirming that AdGuard Home starts correctly.

## Build

Build on FreeBSD or pfSense:

```sh
sh build.sh
```

The default output is:

```text
dist/pfSense-pkg-adguardhome.pkg
```

The build script first looks for:

```text
src/usr/local/bin/AdGuardHome_freebsd_amd64.tar.gz
```

If the local asset is missing, it downloads the official AdGuard Home FreeBSD amd64 release:

```text
https://static.adguard.com/adguardhome/release/AdGuardHome_freebsd_amd64.tar.gz
```

## Install

```sh
pkg add -f dist/pfSense-pkg-adguardhome.pkg
service adguardhome start
```

After the first start, open:

```text
http://<pfsense-host>:3000/
```

If Unbound is still using port `53`, set the AdGuard Home DNS listen port to `5353` or another free port during the first-run wizard. After the wizard is complete, switch to the production DNS flow.

## Take Over Port 53

Stop AdGuard Home first:

```sh
service adguardhome stop
```

In the pfSense Web UI, open:

```text
Services > DNS Resolver > General Settings
```

Change the DNS Resolver listen port from `53` to `5353`, then save and apply. At minimum, make sure `127.0.0.1:5353` is available for AdGuard Home upstream queries.

Then configure AdGuard Home with:

```yaml
dns:
  bind_hosts:
    - 0.0.0.0
  port: 53
  upstream_dns:
    - 127.0.0.1:5353
```

Restart the services:

```sh
service unbound restart
service adguardhome start
```

## Verify

Check listening ports:

```sh
sockstat -4 -l | egrep ':(53|5353|3000)'
```

Expected result:

- `AdGuardHome` listens on `*:53`
- `unbound` listens on `127.0.0.1:5353`
- The AdGuard Home Web UI listens on `*:3000`

Test DNS:

```sh
dig @127.0.0.1 -p 53 example.com
dig @127.0.0.1 -p 5353 example.com
```

## Settings Cannot Be Saved

If the AdGuard Home Web UI cannot save filter lists, DNS upstreams, or other settings, check the log first:

```sh
tail -80 /var/log/adguardhome.log
```

If you see errors like:

```text
requesting https://dns10.quad9.net:443/dns-query: context deadline exceeded
reading from url: Get "https://adguardteam.github.io/..."
```

the configured DoH upstream is not reachable, so AdGuard Home fails when it validates the filter URL during save. In that case, use the local Unbound resolver as the AdGuard Home upstream:

```yaml
dns:
  upstream_dns:
    - 127.0.0.1:5353
  bootstrap_dns:
    - 127.0.0.1:5353
```

You can also fix the current configuration from the shell:

```sh
cp -a /usr/local/etc/adguardhome/AdGuardHome.yaml /usr/local/etc/adguardhome/AdGuardHome.yaml.bak.$(date +%Y%m%d%H%M%S)
perl -0pi -e 's/(  upstream_dns:\n)(?:    - .*\n)+/${1}    - 127.0.0.1:5353\n/s; s/(  bootstrap_dns:\n)(?:    - .*\n)+/${1}    - 127.0.0.1:5353\n/s' /usr/local/etc/adguardhome/AdGuardHome.yaml
service adguardhome restart
```

Then test DNS again:

```sh
dig @127.0.0.1 -p 53 example.com
```

Once DNS returns normally, save the settings again in the AdGuard Home Web UI.

## Roll Back

To restore the default pfSense DNS behavior:

```sh
service adguardhome stop
```

Then change the DNS Resolver port back to `53` in the pfSense Web UI:

```text
Services > DNS Resolver > General Settings
```

Restart Unbound:

```sh
service unbound restart
```

## Uninstall

```sh
pkg delete -y pfSense-pkg-adguardhome
```

## Disclaimer

This is an unofficial community project and is not affiliated with, endorsed by, or supported by the pfSense team. Please review the source code carefully before deployment and use it at your own risk.
