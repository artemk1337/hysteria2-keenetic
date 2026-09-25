# Hysteria 2 on Keenetic: GeoIP split routing

This repository contains a small Entware script for an existing [H-wave](https://github.com/for6to9si/H-wave) installation. Devices assigned to the `Hwave` access policy use their normal connection for Russian destination IPs. Other destinations follow the H-wave route through Hysteria 2.

Read the setup guide in your language:

- [Русский](README.ru.md)
- [English](README.en.md)
- [Español](README.es.md)

The script uses country IP lists from [IPdeny](https://www.ipdeny.com/). Country IP routing does not identify blocked domains. The script contains no VPN credentials.
