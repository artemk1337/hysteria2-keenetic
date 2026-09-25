# Enrutamiento dividido con Hysteria 2 en Keenetic

Este script funciona con una instalación existente de [H-wave](https://github.com/for6to9si/H-wave). Los dispositivos asignados a la política `Hwave` acceden directamente a las IP de destino rusas. Los demás destinos usan la ruta de Hysteria 2. El script descarga las redes rusas de [IPdeny](https://www.ipdeny.com/) y las actualiza cada día.

La decisión se toma por IP de destino, no por una lista de sitios bloqueados. Un sitio ruso alojado en una CDN extranjera puede pasar por la VPN. H-wave solo intercepta los puertos configurados en H-wave; este script no cambia esa lista.

## Requisitos

- Router Keenetic con Entware y H-wave funcionando.
- Acceso a la consola de Entware (`~ #`). No ejecutes estos comandos en la consola KeeneticOS (`(config)>`).
- Espacio libre en `/opt` y acceso a `ipdeny.com` desde el router.

En un Keenetic Giga KN-1012 con H-wave 2.12.2 se verificaron la lista IPv4 y las reglas `iptables` de `nat` y `mangle`. Comprueba IPv6 y el comportamiento tras reiniciar en tu propio router.

## Instalación

Comprueba que Hysteria esté activo:

```sh
/opt/etc/init.d/S96hysteria status
```

En la consola de Entware, instala las dependencias y descarga el script:

```sh
opkg update
opkg install curl ipset
curl -fL https://raw.githubusercontent.com/artemk1337/hysteria2-keenetic/main/S99georoute -o /opt/etc/init.d/S99georoute
chmod 700 /opt/etc/init.d/S99georoute
/opt/etc/init.d/S99georoute start
```

Espera un minuto y revisa las reglas:

```sh
/opt/etc/init.d/S99georoute status
ipset list ru_geo4 | head
iptables -t nat -S hwave | head
iptables -t mangle -S hwave | head
```

El estado debe ser `running`. Ambas cadenas `hwave` deben contener `--match-set ru_geo4 dst -j RETURN` antes de `REDIRECT` o `TPROXY`. Si falta la regla, revisa el registro antes de asignar dispositivos:

```sh
tail -40 /opt/var/log/georoute.log
```

Si IPv6 está activado en H-wave, revísalo también:

```sh
ipset list ru_geo6 | head
ip6tables -t nat -S hwave | head
ip6tables -t mangle -S hwave | head
```

Busca `--match-set ru_geo6 dst -j RETURN` en ambas cadenas. Si falta una cadena IPv6 `hwave`, H-wave no está interceptando IPv6 en esa tabla en ese momento.

## Asignar dispositivos

En la interfaz web de Keenetic, asigna primero la política `Hwave` a un dispositivo. Comprueba un sitio ruso y otro extranjero; después asigna los demás dispositivos. Los dispositivos nuevos también necesitan la política, salvo que hayas configurado su asignación automática en Keenetic.

Puedes mirar los contadores después de abrir los sitios:

```sh
iptables -t nat -L hwave -n -v --line-numbers | head
```

El tráfico a una IP rusa debe incrementar el contador de `ru_geo4`. El tráfico a una IP extranjera debe llegar a `REDIRECT`. Los contadores muestran qué regla recibió paquetes; comprueba el túnel abriendo un sitio desde el dispositivo.

## Revertir

Detén el enrutamiento dividido con:

```sh
/opt/etc/init.d/S99georoute stop
```

H-wave seguirá enviando por Hysteria el tráfico de los dispositivos asignados a su política. Para quitar el arranque automático, borra el archivo después de detenerlo:

```sh
rm /opt/etc/init.d/S99georoute
```

El script revisa sus reglas cada 30 segundos después de reinicios de H-wave o del cortafuegos y actualiza las listas de IP cada día. Si falla una descarga, conserva la lista anterior. Guarda el registro en `/opt/var/log/georoute.log`.
