# VM5 - DNS interno (dnsmasq)

## Función

Servidor DNS interno del laboratorio. Resuelve los nombres de todas las
máquinas (`ws`, `vm1-proxy`, `vm2-app`, `vm3-mon`, `vm4-db`, `vm5-dns`) y
reenvía el resto de consultas al DNS de la red NAT para que sigan
funcionando los nombres de internet.

Se creó para sustituir el mantenimiento manual de `/etc/hosts` en cada
máquina y el `extra_hosts` del `docker-compose.yml` de VM2. Es
infraestructura transversal, igual que VM3: no forma parte de las 3 capas
de la aplicación.

## ¿Por qué una VM propia y no VM3?

- Evita una dependencia circular: Prometheus (VM3) usa nombres para
  monitorizar el resto, y no debería depender de sí mismo para resolverlos.
- Cada máquina de infraestructura tiene una sola responsabilidad.
- Si cae la monitorización, el DNS sigue funcionando, y al revés.

## Especificaciones

| | |
|---|---|
| Distro | Debian (instalación mínima) |
| vCPU | 1 |
| RAM | 1 GB |
| Disco | 10 GB |
| Hostname | vm5-dns |

## Red

| Adaptador | Tipo | Interfaz | IP |
|---|---|---|---|
| 1 | NAT | `ens33` | asignada por DHCP (salida a internet) |
| 2 | Host-only (VMnet1) | `ens34` | 192.168.56.15/24 |

Configurada en `/etc/network/interfaces`:
```
auto ens33
iface ens33 inet dhcp

auto ens34
iface ens34 inet static
    address 192.168.56.15/24
```

## Software instalado

- `dnsmasq`
- `dnsutils` (para `dig`)
- `prometheus-node-exporter`
- `ufw`
- `open-vm-tools`

## Tabla de nombres

dnsmasq sirve los nombres que hay en el `/etc/hosts` de esta máquina. Es el
**único sitio** donde se mantiene la tabla; el resto de máquinas ya no
tienen entradas del laboratorio en su `/etc/hosts`.

```
127.0.0.1       localhost
192.168.56.10   ws
192.168.56.11   vm1-proxy
192.168.56.12   vm2-app
192.168.56.13   vm3-mon
192.168.56.14   vm4-db
192.168.56.15   vm5-dns
```

## Configuración de dnsmasq

`/etc/dnsmasq.d/lab.conf`:
```
listen-address=127.0.0.1,192.168.56.15
bind-interfaces
no-resolv
server=192.168.235.2
```

- `listen-address` + `bind-interfaces`: escucha solo en loopback y en la
  red interna.
- `no-resolv`: no lee el `resolv.conf` del sistema.
- `server`: DNS de la red NAT de VMware, al que se reenvía todo lo que no
  esté en la tabla.

Se valida antes de reiniciar, como se hace con `named-checkzone` en BIND9
o `promtool check config` en Prometheus:
```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

## Firewall

`ufw` activo:
```bash
sudo ufw allow OpenSSH
sudo ufw allow from 192.168.56.0/24 to any port 53
sudo ufw allow from 192.168.56.0/24 to any port 9100
sudo ufw enable
```

## Monitorización

`prometheus-node-exporter` en el puerto 9100, vigilado desde VM3 con un
job nuevo en `prometheus.yml`:
```yaml
  - job_name: 'vm5-dns'
    static_configs:
      - targets: ['vm5-dns:9100']
```

## Configuración del DNS en los clientes

| Máquina | Sistema | Método |
|---|---|---|
| ws | Fedora | `nmcli`: DNS en `host-only`, `ignore-auto-dns` en la NAT, y `ResolveUnicastSingleLabel=yes` en `systemd-resolved` |
| VM1 | Ubuntu | Netplan: `nameservers` en la host-only, `use-dns: false` en la NAT, y `ResolveUnicastSingleLabel=yes` en `systemd-resolved` |
| VM2 | Rocky | `nmcli`, igual que `ws` |
| VM3 | Debian | `dhcpcd`: `resolv.conf.head` + `nooption domain_name_servers` |
| VM4 | Rocky | `nmcli`, igual que `ws` |

Ejemplo con `nmcli` (VM2, VM4 y `ws`):
```bash
sudo nmcli connection modify "<conexión NAT>" ipv4.ignore-auto-dns yes
sudo nmcli connection modify host-only ipv4.dns 192.168.56.15
sudo nmcli connection up "<conexión NAT>"
sudo nmcli connection up host-only
```

## Pasos realizados

1. Instalación de Debian mínimo y configuración de red (NAT + host-only).
2. Creación de la tabla de nombres en `/etc/hosts`.
3. Instalación y configuración de dnsmasq, validada con `dnsmasq --test`.
4. Activación del firewall con acceso al puerto 53 solo desde la red interna.
5. Instalación de `node_exporter` y alta en Prometheus.
6. Pruebas con `dig` desde `ws`:
   ```bash
   dig @192.168.56.15 vm4-db +short
   dig @192.168.56.15 google.com +short
   ```
7. Migración de los clientes de uno en uno, comprobando cada uno con la
   entrada de `vm4-db` comentada en su `/etc/hosts` para asegurar que
   resolvía el DNS y no el archivo.
8. Eliminación de `extra_hosts` en VM2 y comprobación desde el contenedor:
   ```bash
   docker exec wp_app getent hosts vm4-db
   ```
9. Limpieza de las entradas del laboratorio en el `/etc/hosts` de los
   clientes.

## Problemas encontrados

- **`dnsmasq` no arrancaba** (`Name or service not known at line 4`): se
  había copiado la configuración con el texto de ejemplo `server=NAT_DNS`
  sin sustituirlo por la IP real del DNS de la red NAT. Se corrigió con la
  IP correcta (`192.168.235.2`) y se validó con `dnsmasq --test`.
- **Nombres de interfaz distintos en cada VM:** el orden de los adaptadores
  no es el mismo en todas las máquinas (la NAT es `ens33` en VM5 pero
  `ens37` en VM1 y VM3, y `ens160` en VM4). Antes de editar nada hay que
  mirar `ip a` y comprobar qué interfaz tiene qué IP.
- **VM1 (Ubuntu) no resolvía `vm4-db`** aunque el DNS estaba bien
  configurado. `resolvectl query vm4-db` devolvía `No appropriate name
  servers or networks for name found`. La causa: `systemd-resolved` **no
  reenvía a un DNS externo los nombres de una sola etiqueta** (sin punto)
  salvo que se le indique. Se solucionó con:
  ```
  # /etc/systemd/resolved.conf.d/lab.conf
  [Resolve]
  ResolveUnicastSingleLabel=yes
  ```
  Rocky y Debian no tienen este comportamiento porque no usan
  `systemd-resolved` de esta forma.
- **Los contenedores no heredan `/etc/hosts`** (problema original que
  motivó `extra_hosts` en VM2). Con el DNS interno, el contenedor hereda
  el resolvedor del host y `extra_hosts` deja de hacer falta.

## Limitaciones

- VM5 es ahora un **punto único de fallo**: con ella apagada, las
  máquinas siguen funcionando por IP, pero no se resuelven entre sí por
  nombre.
- La tabla de nombres se mantiene a mano en un solo archivo. Siguiente paso
  previsto: gestionarla con Ansible.

## Capturas

![dnsmasq en ejecución](img/vm5-dnsmasq-status.png)
![Consultas dig desde ws a vm5-dns](img/vm5-dig-desde-ws.png)
![Contenedor de WordPress resolviendo vm4-db sin extra_hosts](img/vm2-docker-dns.png)
