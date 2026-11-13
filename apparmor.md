# Playbook de AppArmor · Alba Aérea

Proyecto Drones (TFG) · nodo del centro (tmqdmb) · 13 de noviembre de 2026

## 1. ¿Qué es AppArmor y qué hemos hecho?

AppArmor es una protección que trae el kernel de Linux. Funciona como un guardia: cada programa tiene una lista de lo que puede hacer (qué ficheros toca, qué cosas monta, qué permisos usa) y si intenta algo que no está en la lista, lo bloquea.

En nuestro Proxmox del centro ya viene activado de serie para los contenedores LXC. **No hemos escrito perfiles a mano**: usamos los que genera Proxmox. Pero cada contenedor corre confinado (en modo *enforce*, que significa que bloquea de verdad, no solo avisa) y todos son contenedores sin privilegios.

Este documento explica qué hemos comprobado, cómo se verifica, qué hacer si algo se bloquea y qué reglas seguimos para no debilitar la seguridad.

Última comprobación en el nodo: **13 de noviembre de 2026**.

**Cómo encaja en el proyecto:** AppArmor es una de las capas de seguridad del nodo del centro de Alba Aérea. Protege los siete contenedores (CT101 a CT107): el firewall, la API, la base de datos, la monitorización, el DNS, el proxy web y el túnel de `panel.albaerea.es`. Cuando bloquea algo, avisa por el mismo Telegram que usan las otras alertas de Prometheus y Loki. Trabaja junto con los contenedores sin privilegios, el cortafuegos de Proxmox, nftables en fw-prod y SSH solo con clave. Todavía falta pasarlo a la página "Seguridad y acceso" del Archivo técnico de GitBook.

## 2. Lo que ya tenemos (comprobado)

| Qué | Cómo está | Cómo lo comprobamos |
| --- | --- | --- |
| Módulo de AppArmor | Cargado | `aa-status` dice "apparmor module is loaded" |
| Perfiles | 126 cargados, 27 en modo enforce | `aa-status` |
| Un perfil por contenedor | `lxc-101` a `lxc-107`, todos en enforce | `aa-status` |
| Perfil base | `lxc-container-default` cargado | `aa-status` |
| Ningún contenedor sin protección | Ninguno tiene líneas `lxc.apparmor.*` (no hay `unconfined`) | `grep` en `/etc/pve/lxc/*.conf` |
| Contenedores sin privilegios | Los 7 (`unprivileged: 1`) | `grep` en `/etc/pve/lxc/*.conf` |

Si en CT101 a CT105 ves la línea `unprivileged` repetida, no pasa nada: salen de las instantáneas (por ejemplo `antes-fw`) guardadas dentro del mismo fichero.

### Alerta de bloqueos (13 de noviembre de 2026)

Hemos creado una regla en Loki, `AppArmorBloqueoContenedor`, que avisa por Alertmanager cuando el kernel registra un bloqueo de un perfil `lxc-*` en los últimos 5 minutos (`severity: warning`). El fichero está en `/var/lib/loki/rules/fake/apparmor.yml`, dentro del CT104.

Los bloqueos de `lxc-start` por `vgs` y `lvs` que salen al arrancar el nodo los dejamos fuera a propósito, porque son ruido y no un problema.

La probamos con un mensaje simulado y la regla pasó a *firing* (disparada). Lo que todavía no hemos confirmado es que el aviso llegue a Telegram.

La consulta que usa la regla es esta:

```
sum(count_over_time({job="journal", ident="kernel"} |= "apparmor=\"DENIED\"" |= "profile=\"lxc-" [5m])) > 0
```

### Otras capas de seguridad que lo acompañan

1. Contenedores sin privilegios: el root de dentro no es root en el host.
2. AppArmor en cada contenedor (este documento).
3. Cortafuegos de Proxmox entre contenedores (página 10 del Archivo técnico).
4. nftables en fw-prod (CT101), con la salida controlada.
5. SSH solo con clave en los siete contenedores.

## 3. Lo que todavía NO tenemos

- Perfiles de AppArmor hechos a medida para cada servicio (PostgreSQL, Nginx, cloudflared, FastAPI). Ahora usamos el perfil por defecto de Proxmox.
- Comprobar AppArmor en **pve-home**, el nodo de casa. Aún no lo hemos mirado.
- Dejarlo escrito en GitBook.

Las máquinas virtuales (fw-casa, PDM, SITL) no se protegen de esta forma por dentro: AppArmor del host solo vigila el proceso de la propia VM.

## 4. Cómo funciona en Proxmox

- Cuando arranca un contenedor, LXC crea su perfil (`lxc-<id>_</var/lib/lxc>`) a partir del perfil por defecto y lo carga en el kernel. Por eso en `aa-status` sale un perfil por contenedor.
- Con `features: nesting=1` el contenedor puede montar más cosas y el perfil es un poco más permisivo. En nuestro caso **es obligatorio**: probamos a quitarlo en el CT105 el 13 de noviembre y, sin él, los contenedores Debian 12 (systemd 252) arrancan con `logind`, `networkd` y `networking` fallando.
- La configuración de cada contenedor está en `/etc/pve/lxc/<id>.conf`. Si cambiamos algo de AppArmor, hay que reiniciar el contenedor.

## 5. Cómo comprobar que todo sigue bien

Se hace desde `root@tmqdmb`, una línea cada vez.

**Módulo y perfiles**

```bash
aa-status | head -20
```

Debe decir "apparmor module is loaded" y listar `lxc-101` a `lxc-107` en enforce.

**Que ningún contenedor esté sin protección y todos sin privilegios**

```bash
for i in 101 102 103 104 105 106 107; do echo "CT$i: $(grep -E '^lxc.apparmor|^unprivileged' /etc/pve/lxc/$i.conf | sort -u | tr '\n' ' ')"; done
```

Cada línea debe enseñar solo `unprivileged: 1`. Si sale `lxc.apparmor.profile: unconfined`, hay un problema.

**Nesting (debe salir `nesting=1` en los siete)**

```bash
for i in 101 102 103 104 105 106 107; do echo "CT$i: $(pct config $i | grep -E '^features')"; done
```

**Bloqueos recientes**

```bash
journalctl -k --since "7 days ago" | grep -i 'apparmor="DENIED"' | tail -20
```

Si no sale nada, no ha habido bloqueos esta semana. Los de `lxc-start` (`vgs`, `lvs`) al arrancar el nodo son ruido conocido y los ignoramos.

## 6. Qué hacer si algo se bloquea

Síntoma típico: un servicio de un contenedor falla con "permission denied" aunque los permisos de los ficheros estén bien.

1. Comprobamos que sea AppArmor mirando el log del kernel:

```bash
journalctl -k --since "30 min ago" | grep -i 'apparmor="DENIED"' | tail -10
```

2. Leemos tres campos: `profile=` (qué contenedor), `operation=` (qué intentó hacer) y `name=` o `fstype=` (sobre qué).
3. Pensamos si lo que intentaba es legítimo. Si lo es, añadimos **solo esa regla** al contenedor. No desactivamos el perfil entero:

```bash
# ejemplo, solo si el bloqueo era un montaje legítimo
echo 'lxc.apparmor.raw: mount fstype=<tipo>,' >> /etc/pve/lxc/<id>.conf
pct reboot <id>
```

4. Antes de tocar la configuración hacemos una copia (`cp /etc/pve/lxc/<id>.conf /root/<id>.conf.bak-aa`) y apuntamos el cambio en GitBook.

Nunca usamos `lxc.apparmor.profile: unconfined`, ni siquiera "un momento para probar".

## 7. Reglas que seguimos en el proyecto

- Todo contenedor nuevo se crea con `--unprivileged 1`.
- Ningún contenedor lleva `lxc.apparmor.profile: unconfined`.
- `nesting=1` en los siete contenedores: hace falta en Debian 12 (systemd 252) y no se quita.
- Si añadimos una regla con `lxc.apparmor.raw`, la documentamos: qué contenedor, qué fecha y qué bloqueo la causó.
- Cuando creamos, restauramos o clonamos un contenedor, repetimos las comprobaciones de la sección 5.

## 8. Lista de comprobación para un contenedor nuevo

- [ ] Creado con `--unprivileged 1`
- [ ] `nesting` activado (hace falta con systemd 252, ver sección 4)
- [ ] Al arrancar, aparece `lxc-<id>` en `aa-status` en modo enforce
- [ ] Sin líneas `lxc.apparmor.*` en `/etc/pve/lxc/<id>.conf`
- [ ] Sin bloqueos `DENIED` durante la primera hora
- [ ] Apuntado en la ficha del contenedor en GitBook

## 9. Cómo montarlo desde cero (comandos)

Estos son los pasos en el orden en que se hacen en un nodo Proxmox. Los comandos de 9.3 a 9.6 son los que ejecutamos y comprobamos en el proyecto. El de 9.2 sigue el mismo patrón con el que creamos los contenedores del centro, pero adaptado a lo que hemos aprendido (una sola tarjeta de red y `nesting=1`). Ese no lo hemos ejecutado tal cual.

### 9.1 Primero: comprobar que AppArmor está activo en el nodo

```bash
aa-status | head -3
```

Debe decir "apparmor module is loaded". En Proxmox VE viene activo de serie. Solo si el comando no existe: `apt install -y apparmor apparmor-utils` y reiniciar el nodo.

### 9.2 Crear un contenedor protegido

Sin privilegios, con `nesting=1` y una sola tarjeta de red. Si creamos el contenedor con una segunda tarjeta y luego la quitamos, quedan restos en la configuración de red y `networking.service` falla. Y la plantilla siempre amd64 (en la lista también salen las de arm64 y no valen).

```bash
ID=108
pveam list local | grep amd64
pct create $ID local:vztmpl/<plantilla-debian-12-amd64.tar.zst> --hostname <nombre> --unprivileged 1 --features nesting=1 --cores 1 --memory 512 --swap 256 --rootfs local-lvm:4 --net0 name=eth0,bridge=vmbr1,firewall=1,ip=10.10.11.<x>/24,gw=10.10.11.10,type=veth --nameserver 10.10.11.50 --onboot 1
pct start $ID
```

No añadimos ninguna línea `lxc.apparmor.*`: usamos el perfil que genera Proxmox.

### 9.3 Comprobar que el contenedor está protegido

```bash
sleep 20
aa-status | grep lxc-$ID
grep -E '^lxc.apparmor|^unprivileged|^features' /etc/pve/lxc/$ID.conf
pct exec $ID -- systemctl --failed --no-legend
journalctl -k --since "10 min ago" | grep 'apparmor="DENIED"' | grep "profile=\"lxc-$ID"
```

Tiene que salir: `lxc-<id>` en la lista, solo `unprivileged: 1` y `features: nesting=1`, ningún servicio fallido (menos `openipmi` y `nvmf-autoconnect`, que se ignoran porque en un contenedor no sirven para nada) y ningún `DENIED`. Después hacemos una instantánea limpia: `pct snapshot $ID limpio`.

### 9.4 Crear la alerta de bloqueos en Loki (una vez por nodo)

Lo que hace falta y ya tenemos en el CT104: el promtail del nodo lee el journal con `job: journal` y saca la etiqueta `ident` del identificador del syslog, y el evaluador de reglas de Loki tiene `alertmanager_url` y guarda las reglas en `/var/lib/loki/rules`.

```bash
cat > /root/apparmor.yml <<'EOF'
groups:
  - name: apparmor
    rules:
      - alert: AppArmorBloqueoContenedor
        expr: sum(count_over_time({job="journal", ident="kernel"} |= "apparmor=\"DENIED\"" |= "profile=\"lxc-" [5m])) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "AppArmor ha bloqueado una operacion de un contenedor"
          description: "Bloqueos de AppArmor en un perfil lxc-* en los ultimos 5 minutos. En tmqdmb: journalctl -k | grep DENIED"
EOF
pct push 104 /root/apparmor.yml /var/lib/loki/rules/fake/apparmor.yml --perms 0644
sleep 70
pct exec 104 -- curl -s http://10.10.11.40:3100/prometheus/api/v1/rules | grep -o '"state":"[a-z]*","name":"AppArmorBloqueoContenedor"'
```

Tiene que salir `"state":"inactive"` (cargada y sin bloqueos). Para comprobar que la consulta encuentra bloqueos de la última hora (devuelve un número):

```bash
pct exec 104 -- curl -s -G http://10.10.11.40:3100/loki/api/v1/query --data-urlencode 'query=sum(count_over_time({job="journal", ident="kernel"} |= "apparmor=\"DENIED\"" |= "profile=\"lxc-" [1h]))'
```

### 9.5 Probar la alerta de principio a fin

Un mensaje escrito a mano en el log del kernel llega a Loki sin la etiqueta `ident`. Por eso, mientras probamos, quitamos ese filtro y luego lo volvemos a poner.

```bash
pct exec 104 -- sed -i 's/{job="journal", ident="kernel"}/{job="journal"}/' /var/lib/loki/rules/fake/apparmor.yml
sleep 70
echo 'PRUEBA audit: apparmor="DENIED" operation="prueba" profile="lxc-999_</var/lib/lxc>" comm="test"' > /dev/kmsg
sleep 100
pct exec 104 -- curl -s http://10.10.11.40:3100/prometheus/api/v1/rules | grep -o '"state":"[a-z]*","name":"AppArmorBloqueoContenedor"'
```

Tiene que pasar a `firing` y llegar el aviso a Telegram. En la prueba del 13 de noviembre pasó a `firing`, pero el aviso de Telegram todavía no lo hemos confirmado. Al acabar, **siempre** volvemos a poner el filtro estricto:

```bash
pct exec 104 -- sed -i 's/{job="journal"}/{job="journal", ident="kernel"}/' /var/lib/loki/rules/fake/apparmor.yml
```

### 9.6 Arreglar un contenedor con restos de una segunda tarjeta de red

Síntoma: `networking.service` aparece fallido y en el journal del contenedor sale `ifup: failed to bring up eth1`. Quitamos las entradas viejas sin reiniciar la red y con copia de seguridad:

```bash
ID=105
pct exec $ID -- cp /etc/network/interfaces /etc/network/interfaces.bak-aa
pct exec $ID -- sed -i '/^auto \(eth1\|net0\|net1\)$/d;/^iface \(eth1\|net0\|net1\) inet dhcp$/d' /etc/network/interfaces
pct exec $ID -- systemctl reset-failed networking.service ifupdown-wait-online.service
pct exec $ID -- grep -E '^(auto|iface)' /etc/network/interfaces
```

Al final tiene que quedar solo `lo` y `eth0`. El arreglo se nota del todo en el siguiente arranque del contenedor.

## 10. Registro de lo que hemos hecho

- 10 nov 2026: comprobamos AppArmor en el nodo del centro: módulo cargado, 126 perfiles, 27 en enforce, `lxc-101` a `lxc-107` en enforce, sin `unconfined` y los siete contenedores sin privilegios. Es la primera vez que lo dejamos documentado.
- 13 nov 2026: probamos a quitar `nesting` en el CT105. Fallaron `networking`, `systemd-networkd`, `systemd-logind` y `openipmi`, y el perfil `lxc-105` bloqueó montajes de `proc` y `ramfs` (6 bloqueos). Volvimos atrás con la instantánea. Conclusión: `nesting=1` hace falta con Debian 12 (systemd 252) y los siete lo llevan.
- 13 nov 2026: no hay bloqueos de perfiles `lxc-*` aparte de los de la prueba. Los 10 `DENIED` de `lxc-start` (`vgs`, `lvs`) del arranque del 10 nov son ruido.
- 13 nov 2026: quitamos las entradas viejas `eth1`, `net0` y `net1` de `/etc/network/interfaces` en CT102 a CT105 (con copia `interfaces.bak-aa`). Eran la causa de que `networking.service` apareciera fallido. `openipmi` y `nvmf-autoconnect` siguen fallando en varios contenedores, pero no afecta a nada.
- 13 nov 2026: creamos la alerta `AppArmorBloqueoContenedor` en Loki y la probamos hasta `firing`. Falta confirmar que el aviso llega a Telegram.
