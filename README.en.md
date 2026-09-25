# Hysteria 2 split routing on Keenetic

This script works with an existing [H-wave](https://github.com/for6to9si/H-wave) installation. For devices assigned to the `Hwave` access policy, Russian destination IPs use the normal connection; other destinations follow the Hysteria 2 route. The script downloads Russian IP ranges from [IPdeny](https://www.ipdeny.com/) and refreshes them daily.

Routing is based on destination IP, not a list of blocked websites. A Russian site hosted on a foreign CDN may use the VPN. H-wave intercepts only the ports configured in H-wave; this script does not change that port list.

## Requirements

- Keenetic router with Entware and a working H-wave installation.
- Entware shell (`~ #`). Do not run these shell commands in the KeeneticOS CLI (`(config)>`).
- Free space on `/opt` and access to `ipdeny.com` from the router.

On a Keenetic Giga KN-1012 with H-wave 2.12.2, the IPv4 list and `iptables` rules in `nat` and `mangle` were verified. Check IPv6 and reboot behavior on your own router.

## Install

Check that Hysteria is running:

```sh
/opt/etc/init.d/S96hysteria status
```

In the Entware shell, install dependencies and download the script:

```sh
opkg update
opkg install curl ipset
curl -fL https://raw.githubusercontent.com/artemk1337/hysteria2-keenetic/main/S99georoute -o /opt/etc/init.d/S99georoute
chmod 700 /opt/etc/init.d/S99georoute
/opt/etc/init.d/S99georoute start
```

Wait one minute, then inspect the rules:

```sh
/opt/etc/init.d/S99georoute status
ipset list ru_geo4 | head
iptables -t nat -S hwave | head
iptables -t mangle -S hwave | head
```

The status should be `running`. Both `hwave` chains should have `--match-set ru_geo4 dst -j RETURN` before `REDIRECT` or `TPROXY`. If the rule is missing, inspect the log before assigning devices to the policy:

```sh
tail -40 /opt/var/log/georoute.log
```

If IPv6 is enabled in H-wave, check it too:

```sh
ipset list ru_geo6 | head
ip6tables -t nat -S hwave | head
ip6tables -t mangle -S hwave | head
```

Look for `--match-set ru_geo6 dst -j RETURN` in both chains. A missing IPv6 `hwave` chain means H-wave is not intercepting IPv6 in that table at the moment.

## Assign devices

In the Keenetic web UI, assign the `Hwave` access policy to one device first. Check a Russian and a foreign website on that device, then assign the remaining devices. New devices also need the policy unless you have configured an automatic assignment rule in Keenetic.

You can inspect rule counters after opening the websites:

```sh
iptables -t nat -L hwave -n -v --line-numbers | head
```

Traffic to a Russian IP should increment the `ru_geo4` rule. Traffic to a foreign IP should reach `REDIRECT`. Counters show which rule received traffic; confirm that the tunnel works by opening a site on the client device.

## Roll back

Stop split routing with:

```sh
/opt/etc/init.d/S99georoute stop
```

H-wave will keep routing devices assigned to its policy through Hysteria. To disable the script at boot, remove it after stopping:

```sh
rm /opt/etc/init.d/S99georoute
```

The script checks its rules every 30 seconds after H-wave or firewall restarts and refreshes IP lists daily. If a download fails, it keeps the previous list. Logs are stored in `/opt/var/log/georoute.log`.
