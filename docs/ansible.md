# Automatización con Ansible

## Objetivo

Pasar a código todo lo que se montó a mano en el laboratorio, para que se
pueda repetir, revisar y comprobar. Ansible se ejecuta desde `ws` (el nodo
de control) y administra por SSH las 5 VMs de servidor.

Qué está automatizado:

| Playbook | Rol | Máquinas | Qué hace |
|---|---|---|---|
| `base.yml` | (tareas) | todas | paquetes comunes |
| `dns.yml` | `dnsmasq` | VM5 | servidor DNS interno |
| `dns_client.yml` | `dns_client` | VM1-VM4 | uso del DNS interno en cada cliente |
| `node_exporter.yml` | `node_exporter` | todas | métricas para Prometheus |
| `monitoring.yml` | `prometheus`, `grafana` | VM3 | Prometheus y Grafana (fuente de datos y dashboard como código) |
| `proxy.yml` | `nginx` | VM1 | proxy inverso con HTTPS |
| `docker.yml` | `docker` | VM2, VM4 | Docker CE y Compose |
| `apps.yml` | `mariadb`, `wordpress` | VM4, VM2 | base de datos y aplicación, con credenciales en Vault |
| `site.yml` | (todos) | todas | el laboratorio completo, en orden |

El código está en la carpeta [`ansible/`](../ansible).

## Requisitos

En `ws`:

```bash
sudo dnf install -y ansible
ansible --version
ansible-galaxy collection list community.general
ansible-galaxy collection list community.docker
```

El paquete `ansible` de Fedora ya incluye las colecciones
`community.general` (módulo `ufw`) y `community.docker` (módulo
`docker_compose_v2`). Si faltara alguna:
```bash
ansible-galaxy collection install community.general community.docker
```

Además:
- Acceso SSH sin contraseña desde `ws` a todas las VMs (ver
  `docs/workstation-fedora.md`).
- El usuario `piiter` con permisos de `sudo` en todas las VMs.
- En Ubuntu (VM1), el `sudo` clásico en lugar de `sudo-rs` (ver
  **Problemas encontrados**).

## Estructura

```
ansible/
├── ansible.cfg
├── inventory.ini
├── site.yml                  # todos los playbooks, en orden
├── base.yml                  # paquetes comunes
├── dns.yml                   # servidor DNS (VM5)
├── dns_client.yml            # DNS interno en los clientes (VM1-VM4)
├── node_exporter.yml         # métricas en todas las VMs
├── monitoring.yml            # Prometheus y Grafana (VM3)
├── proxy.yml                 # Nginx (VM1)
├── docker.yml                # Docker (VM2 y VM4)
├── apps.yml                  # MariaDB (VM4) y WordPress (VM2)
├── host_vars/                # interfaces y método de red de cada VM
│   ├── vm1-proxy.yml
│   ├── vm2-app.yml
│   ├── vm3-mon.yml
│   └── vm4-db.yml
├── group_vars/lab/
│   ├── vars.yml              # variables que apuntan al Vault
│   └── vault.yml.example     # plantilla; el vault.yml real no se sube
└── roles/
    ├── dnsmasq/              # defaults, tasks, handlers, templates
    ├── dns_client/           # tasks: main, nmcli, netplan y dhcpcd
    ├── node_exporter/        # tasks: main, debian y redhat
    ├── prometheus/
    ├── grafana/
    ├── nginx/
    ├── docker/
    ├── mariadb/
    └── wordpress/
```

Cada rol sigue la estructura estándar:

| Carpeta | Contenido |
|---|---|
| `defaults/` | variables con valores por defecto (lo que se puede cambiar) |
| `tasks/` | lo que hace el rol |
| `handlers/` | acciones que solo se ejecutan si algo cambia (reinicios, recargas) |
| `templates/` | archivos de configuración con variables (Jinja2) |

## Configuración

### `ansible.cfg`
```ini
[defaults]
inventory = ./inventory.ini
interpreter_python = auto_silent
```

### Inventario
Un grupo por función, todos agrupados en `lab`. Los nombres los resuelve
el DNS interno (ver `docs/vm5-dns.md`), así que no hay IPs:

```ini
[proxy]
vm1-proxy

[app]
vm2-app

[monitor]
vm3-mon

[db]
vm4-db

[dns]
vm5-dns

[lab:children]
proxy
app
monitor
db
dns

[lab:vars]
ansible_user=piiter
```

`ws` no está en el inventario: es la máquina desde la que se ejecuta
Ansible, no uno de sus destinos.

### Variables por máquina (`host_vars/`)
Cada distribución gestiona la red de una forma distinta y los nombres de
interfaz cambian de una VM a otra, así que esos datos van en un archivo
por máquina. Ejemplo (`host_vars/vm2-app.yml`):

```yaml
---
dns_client_method: nmcli
dns_client_nat_conn: "Wired connection 1"
dns_client_nat_iface: ens224
dns_client_host_conn: host-only
dns_client_host_iface: ens160
```

Antes de escribirlos se comprobaron los nombres reales en cada máquina:
```bash
ssh vm1-proxy "ls -l /etc/netplan/"
ssh vm2-app "nmcli -t -f NAME,DEVICE connection show --active"
ssh vm3-mon "ip -br a"
ssh vm4-db "nmcli -t -f NAME,DEVICE connection show --active"
```

### Credenciales con Ansible Vault
Las credenciales de la base de datos están en
`group_vars/lab/vault.yml`, cifrado con `ansible-vault`. `vars.yml` las
expone con los nombres que usan las plantillas:

```yaml
---
mysql_database: "{{ vault_mysql_database }}"
mysql_user: "{{ vault_mysql_user }}"
mysql_password: "{{ vault_mysql_password }}"
mysql_root_password: "{{ vault_mysql_root_password }}"
```

El `vault.yml` real **no se sube al repositorio** (está en
`.gitignore`). Solo se publica `vault.yml.example` con la estructura:

```bash
cp group_vars/lab/vault.yml.example group_vars/lab/vault.yml
vim group_vars/lab/vault.yml                   # valores reales
ansible-vault encrypt group_vars/lab/vault.yml
ansible-vault view group_vars/lab/vault.yml    # consultarlo
ansible-vault edit group_vars/lab/vault.yml    # cambiarlo
```

Como el Vault está en `group_vars/lab/`, se carga para todas las
máquinas: **cualquier playbook necesita `--ask-vault-pass`**, aunque no
use las credenciales.

## Flujo de trabajo

Cada playbook se ha aplicado siempre con los mismos pasos:

```bash
cd ~/lab-ansible

# 1. Conectividad y permisos
ansible lab -m ping
ansible lab -b -K --ask-vault-pass -m command -a "whoami"

# 2. Sintaxis
ansible-playbook <playbook>.yml --syntax-check

# 3. Simulacro: qué cambiaría, sin tocar nada
ansible-playbook <playbook>.yml --check --diff -K --ask-vault-pass

# 4. Ejecución real
ansible-playbook <playbook>.yml -K --ask-vault-pass

# 5. Repetir: debe dar changed=0 (idempotencia)
ansible-playbook <playbook>.yml -K --ask-vault-pass
```

- `-K` pide la contraseña de `sudo`. No se guarda en ningún archivo.
- `--check --diff` enseña línea a línea qué cambiaría en cada archivo. Es
  el paso más importante: nada se ejecuta en real sin revisar antes el
  diff.
- Para limitar un playbook a una sola máquina: `--limit vm1-proxy`.

## Playbooks

### `base.yml`
Instala `curl`, `htop`, `tmux`, `tree`, `jq` y `vim` en todas las VMs. El
paquete de `vim` se llama `vim-enhanced` en Rocky y `vim` en Debian y
Ubuntu, y el playbook elige el nombre según la familia de la
distribución. La caché de `apt` solo se actualiza en Debian y Ubuntu.

```bash
ansible-playbook base.yml --check --diff -K --ask-vault-pass
ansible-playbook base.yml -K --ask-vault-pass
```

![Ejecución de base.yml con cambios y repetición sin cambios](img/ansible-base-idempotencia.png)

### `dns.yml` (rol `dnsmasq`)
Configura el servidor DNS de VM5. La tabla de nombres está en una
variable (`dnsmasq_hosts`) y se genera con una plantilla, de modo que la
fuente de verdad es el repositorio y no el archivo de la máquina.

- La configuración se **valida antes de guardarse** con
  `dnsmasq --test -C`, igual que se haría con `named-checkzone` en BIND9.
- Un *handler* reinicia dnsmasq solo cuando cambia algo.

```bash
ansible-playbook dns.yml --check --diff -K --ask-vault-pass
ansible-playbook dns.yml -K --ask-vault-pass
dig @192.168.56.15 vm4-db +short      # 192.168.56.14
```

![Ejecución de dns.yml con cambios, repetición sin cambios y consulta dig a vm4-db](img/ansible-dnsmasq-idempotencia.png)

### `dns_client.yml` (rol `dns_client`)
Hace que VM1-VM4 usen el DNS interno (VM5 es el propio servidor, por eso
el playbook usa `hosts: lab:!dns`). Cada distribución lo hace de una
forma, así que el rol lee `dns_client_method` de `host_vars/` e incluye
las tareas que tocan:

| Método | Máquinas | Qué configura |
|---|---|---|
| `nmcli` | VM2, VM4 (Rocky) | `ipv4.ignore-auto-dns yes` en la NAT e `ipv4.dns` en la host-only |
| `netplan` | VM1 (Ubuntu) | plantilla de Netplan y `ResolveUnicastSingleLabel=yes` en `systemd-resolved` |
| `dhcpcd` | VM3 (Debian) | `/etc/resolv.conf.head` y `nooption domain_name_servers` |

```yaml
# roles/dns_client/tasks/main.yml
- name: Comprobar que la máquina tiene un método de DNS definido
  ansible.builtin.assert:
    that: dns_client_method in ['nmcli', 'netplan', 'dhcpcd']

- name: Configurar el DNS con el método de la distribución
  ansible.builtin.include_tasks: "{{ dns_client_method }}.yml"
```

Decisiones de diseño:
- **`nmcli` idempotente.** Un `nmcli connection modify` siempre da
  `changed`. El rol primero **lee** el valor actual (`nmcli -g`, con
  `changed_when: false` y `check_mode: false` para que también funcione en
  simulacro) y solo **modifica** si es distinto.
- **Sin cortar el SSH.** Los cambios se aplican con
  `nmcli device reapply`, que no baja la interfaz. Con
  `nmcli connection up`, como se hizo a mano, se cortaría la conexión por
  la que trabaja Ansible.
- **`lineinfile` en vez de `echo >>`**, para que la línea de `dhcpcd.conf`
  no se duplique en cada ejecución.

```bash
ansible-playbook dns_client.yml --syntax-check
ansible-playbook dns_client.yml --check --diff -K --ask-vault-pass
ansible-playbook dns_client.yml -K --ask-vault-pass
ansible-playbook dns_client.yml -K --ask-vault-pass   # changed=0

ssh vm1-proxy "resolvectl query vm4-db; ls -l /etc/netplan/"
```

Resultado del simulacro: VM2, VM3 y VM4 con `changed=0` (ya estaban
configuradas igual que el rol), y en VM1 solo la cabecera
`# Gestionado por Ansible` en el archivo de Netplan y el cambio de
permisos a `0600`, que es lo que exige Netplan.

### `node_exporter.yml` (rol `node_exporter`)
Instala y activa `node_exporter` en todas las VMs, con un caso por
familia de distribución:

- **Debian y Ubuntu** (VM1, VM3, VM5): paquete `prometheus-node-exporter`.
- **Rocky** (VM2, VM4): binario oficial en una versión fijada
  (`node_exporter_version`), usuario de servicio, unidad systemd desde una
  plantilla y puerto 9100 en `firewalld`. Si la versión instalada ya
  coincide, no se descarga nada.

```bash
ansible-playbook node_exporter.yml --check --diff -K --ask-vault-pass
ansible-playbook node_exporter.yml -K --ask-vault-pass
```

![node_exporter.yml sin cambios en las 5 VMs](img/ansible-node-exporter-recap.png)

### `monitoring.yml` (rol `prometheus`)
Genera `/etc/prometheus/prometheus.yml` en VM3 a partir del inventario.
Los jobs no se escriben a mano: un bucle crea uno por cada máquina del
grupo `lab`, así que al añadir una VM al inventario Prometheus la vigila
sin tocar nada más.

```yaml
# roles/prometheus/defaults/main.yml
prometheus_targets: "{{ groups['lab'] | difference([inventory_hostname]) | sort }}"
```

```jinja
{# roles/prometheus/templates/prometheus.yml.j2 #}
{% for host in prometheus_targets %}
  - job_name: '{{ host }}'
    static_configs:
      - targets: ['{{ host }}:{{ prometheus_node_port }}']
{% endfor %}
```

- `difference([inventory_hostname])` quita la propia VM3, que ya está en
  el job `node` (`localhost`).
- `sort` ordena la lista para que el archivo generado sea siempre igual;
  sin él, el orden podía variar y la tarea daría `changed` en cada
  ejecución.
- El archivo se valida con `promtool check config` **antes** de sustituir
  el actual (`validate:`). Si tuviera un error, no se guarda y Prometheus
  sigue con el anterior: el fallo que tumbó el servicio al principio del
  laboratorio ya no se puede repetir.
- `backup: true` deja una copia del archivo anterior en VM3.

```bash
ansible-playbook monitoring.yml --check --diff -K --ask-vault-pass
ansible-playbook monitoring.yml -K --ask-vault-pass
ansible-playbook monitoring.yml -K --ask-vault-pass   # changed=0

curl -s http://vm3-mon:9090/api/v1/targets | grep -o '"health":"[a-z]*"' | sort | uniq -c
#   6 "health":"up"
```

El diff del simulacro mostró que el único cambio real era la cabecera de
ejemplo que trae Debian (`external_labels`, un `alerting` hacia un
Alertmanager que no está instalado y `rule_files` vacío). Los seis jobs
quedaron idénticos. Justo tras el reinicio, 3 targets salen como
`unknown` hasta su primera lectura; a los 15-20 segundos están los 6 en
`up`.


### `monitoring.yml` (rol `grafana`)
La fuente de datos y el dashboard de Grafana se habían creado desde la
web, así que vivían en su base de datos interna: si VM3 se rompía, se
perdían. El rol los pasa a código con el *provisioning* de Grafana, que
lee archivos YAML de `/etc/grafana/provisioning/` al arrancar.

Antes de escribirlo se revisó lo que ya existía, para que el rol lo
adoptara en vez de duplicarlo:

```bash
ssh vm3-mon "cat /etc/apt/sources.list.d/grafana.list"
curl -s -u admin http://vm3-mon:3000/api/datasources | python3 -m json.tool
curl -s -u admin "http://vm3-mon:3000/api/search?type=dash-db" | python3 -m json.tool
```

Qué hace:
- Escribe el repositorio de Grafana con el mismo archivo y la misma clave
  que ya había (`/etc/apt/keyrings/grafana.gpg`), para que `apt` no lo vea
  duplicado.
- **Fuente de datos** (`provisioning/datasources/lab.yml`): crea
  `prometheus` con el **mismo `uid`** que la creada a mano, de modo que el
  dashboard importado no pierde los datos. La marca como predeterminada y
  borra una fuente `graphite` vacía que estaba como predeterminada por
  error.
- **Dashboards** (`provisioning/dashboards/lab.yml`): carga todos los JSON
  de `/var/lib/grafana/dashboards` en la carpeta *Laboratorio*. El rol
  descarga de grafana.com los de la lista `grafana_dashboards` (ahora el
  *Node Exporter Full*, ID 1860). Como tiene el mismo `uid` que el
  importado a mano, Grafana lo adopta en vez de duplicarlo.
- La descarga lleva `force: false`, para que solo se haga una vez y la
  tarea sea idempotente.

```bash
ansible-playbook monitoring.yml --check --diff -K --ask-vault-pass
ansible-playbook monitoring.yml -K --ask-vault-pass
ansible-playbook monitoring.yml -K --ask-vault-pass   # changed=0

curl -s -u admin http://vm3-mon:3000/api/datasources | python3 -m json.tool | grep -E '"(name|uid|isDefault|readOnly)"'
curl -s -u admin "http://vm3-mon:3000/api/search?type=dash-db" | python3 -m json.tool | grep -E '"(uid|title|folderTitle)"'
```

Resultado: una sola fuente de datos (`prometheus`, predeterminada y
`readOnly: true`, es decir, gestionada por archivo) y un solo dashboard,
dentro de la carpeta *Laboratorio*. Desde la web ya no se pueden
modificar: la fuente de verdad son los archivos del repositorio.

### `proxy.yml` (rol `nginx`)
Configura VM1 como proxy inverso: instala Nginx, genera el sitio desde
una plantilla, lo activa, desactiva el sitio por defecto, aplica las
reglas de `ufw` e instala el *drop-in* de systemd que hace esperar a
Nginx hasta que haya red y DNS.

```nginx
# roles/nginx/templates/proxy.j2 (extracto)
location / {
    proxy_pass {{ nginx_backend }};
    proxy_set_header Host $host;
    ...
}
```

Las variables de Ansible van entre `{{ }}` y se rellenan al generar el
archivo; las de Nginx (`$host`, `$scheme`) se quedan tal cual.

Decisiones de diseño:
- **Certificado:** el comando `openssl` lleva `creates:`, así que solo se
  ejecuta si la clave no existe. El certificado actual no se toca, y en
  una máquina nueva se generaría solo.
- **Validar antes de recargar:** cualquier cambio avisa a dos *handlers*
  que escuchan el mismo aviso y se ejecutan en orden:
  ```yaml
  - name: Comprobar la configuración de Nginx
    ansible.builtin.command: nginx -t
    changed_when: false
    listen: Validar Nginx

  - name: Recargar Nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
    listen: Validar Nginx
  ```
  Si `nginx -t` falla, la recarga no llega a hacerse y Nginx sigue
  sirviendo con la configuración anterior. Se usa `reloaded` y no
  `restarted` para no cortar las conexiones abiertas.

```bash
ansible-playbook proxy.yml --syntax-check
ansible-playbook proxy.yml --check --diff -K --ask-vault-pass
ansible-playbook proxy.yml -K --ask-vault-pass
ansible-playbook proxy.yml -K --ask-vault-pass        # changed=0

curl -kI https://vm1-proxy                             # HTTP/1.1 200 OK
ssh vm1-proxy "systemctl show nginx -p DropInPaths"
# DropInPaths=/etc/systemd/system/nginx.service.d/espera-dns.conf
```

El diff del simulacro solo mostró comentarios: la cabecera
`# Gestionado por Ansible` y la desaparición de un comentario escrito a
mano. Certificado, enlaces, servicio y `ufw` salieron `ok`, confirmando
que coincidían con lo configurado manualmente.

### `docker.yml` (rol `docker`)
Instala Docker CE y el plugin de Compose en VM2 y VM4 desde el
repositorio oficial: importa la clave GPG (`rpm_key`), descarga el
`.repo`, instala los paquetes de `docker_packages`, activa el servicio y
añade a los usuarios de `docker_users` al grupo `docker`.

```bash
ansible-playbook docker.yml --check --diff -K --ask-vault-pass
ansible-playbook docker.yml -K --ask-vault-pass
```

### `apps.yml` (roles `mariadb` y `wordpress`)
Despliega las dos aplicaciones con `docker compose`, primero la base de
datos (VM4) y después WordPress (VM2). Cada rol genera el
`docker-compose.yml` y el `.env` desde plantillas, y levanta el proyecto
con `community.docker.docker_compose_v2`.

- El `.env` se crea con permisos `0600`, `backup: true`, y `no_log` y
  `diff: false` para que las contraseñas nunca aparezcan en la salida.
- WordPress no necesita la contraseña de `root` de la base de datos, así
  que su `.env` solo lleva tres variables.
- El proyecto de Compose toma el nombre del directorio. Como el directorio
  es el mismo que se usaba a mano, **los volúmenes se conservan** y los
  datos no se pierden al pasar a Ansible.

```bash
# Copia de seguridad de la base de datos antes de cualquier ejecución real
ssh vm4-db "docker exec wp_db mariadb-dump -u<usuario> -p<contraseña> wordpress" > ~/wordpress-backup.sql

ansible-playbook apps.yml --check --diff -K --ask-vault-pass
ansible-playbook apps.yml -K --ask-vault-pass
ansible-playbook apps.yml -K --ask-vault-pass         # changed=0

ssh vm4-db "docker ps --format '{{.Names}} {{.Status}}'"
curl -kI https://vm1-proxy
```

### `site.yml`
Lanza todos los playbooks en orden de dependencias con `import_playbook`.
Los playbooks individuales siguen sirviendo para aplicar solo una parte.

```yaml
- import_playbook: base.yml
- import_playbook: dns.yml
- import_playbook: dns_client.yml
- import_playbook: node_exporter.yml
- import_playbook: docker.yml
- import_playbook: apps.yml
- import_playbook: proxy.yml
- import_playbook: monitoring.yml
```

1. `base`: paquetes comunes, antes que nada.
2. `dns` y `dns_client`: el servidor y después los clientes, porque el
   resto de máquinas se encuentran por nombre.
3. `node_exporter`: no depende de nada más.
4. `docker` y después `apps`: sin Docker no hay contenedores, y dentro de
   `apps` la base de datos va antes que WordPress.
5. `proxy`: después de la aplicación, porque Nginx necesita resolver
   `vm2-app` para arrancar.
6. `monitoring`: al final, cuando todas las máquinas exponen métricas.

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --check --diff -K --ask-vault-pass
ansible-playbook site.yml -K --ask-vault-pass
```

Con todo ya aplicado, el simulacro de `site.yml` da `changed=0` en las
cinco máquinas: el laboratorio coincide con el código. Las únicas
excepciones esperables son la caché de `apt` (se refresca si tiene más de
una hora) y la descarga del `.repo` de Docker en simulacro (ver
**Problemas encontrados**).

## Idempotencia

Todos los playbooks se pueden ejecutar varias veces sin efectos
secundarios: la primera ejecución con cambios muestra `changed`, y la
siguiente `changed=0`. Para conseguirlo:

- Plantillas y `copy` en vez de comandos que escriben archivos.
- `lineinfile` en vez de `echo >>`.
- Comandos con `creates:` o con una lectura previa y `when`.
- Lecturas marcadas con `changed_when: false`.
- Listas ordenadas (`| sort`) cuando generan archivos.
- *Handlers* para reiniciar o recargar solo cuando algo cambia.

## Problemas encontrados

- **`sudo-rs` en Ubuntu (VM1).** Con `-b -K`, Ansible fallaba solo en VM1
  con `Timeout waiting for privilege escalation prompt`. La causa no era el
  DNS ni la red: Ubuntu usa por defecto `sudo-rs`, cuyo prompt
  (`[sudo: authenticate] Password:`) Ansible no reconoce. Se resolvió
  fijando el `sudo` clásico:
  ```bash
  sudo update-alternatives --set sudo /usr/bin/sudo.ws
  ```
  La pista fue el formato del prompt. La primera hipótesis (resolución del
  nombre de la máquina en `sudo`) era errónea y se descartó revisando
  `/etc/hosts` y la salida detallada (`-vvv`).
- **Modo simulacro (`--check`) y módulos `command`.** El rol
  `node_exporter` fallaba solo en simulacro con `Source ... not found`.
  En `--check`, el módulo `command` no se ejecuta, así que Ansible no podía
  saber qué versión había instalada, daba por hecho que no había ninguna e
  intentaba copiar un binario que no se había descargado. Como la consulta
  de versión es de solo lectura, se marcó con `check_mode: false`. El
  mismo patrón se usa después en el rol `dns_client`.
- **El Vault pedía contraseña en playbooks que no lo usan.** Al estar en
  `group_vars/lab/`, se carga para todas las máquinas. Sin
  `--ask-vault-pass`, cualquier playbook falla con
  `Attempting to decrypt but no vault secrets found`.
- **El `.env` de VM4 tenía las variables desplazadas.** Al pasar los
  despliegues a Ansible se copiaron al Vault los valores del `.env` de
  VM4, y en la primera ejecución real WordPress dejó de conectar
  (`Access denied for user 'wordpress'`, que en realidad era el nombre
  de la base de datos). Comparando el hash de cada valor entre VM2 y VM4
  se vio que en VM4 cada valor estaba una posición corrida respecto a su
  nombre. Pasó desapercibido porque MariaDB solo lee esas variables la
  primera vez que crea la base de datos. Los valores buenos eran los de
  VM2. Se corrigió el Vault, se restauraron los `.env` desde las copias
  que deja `backup: true` y se volvió a ejecutar `apps.yml`.
- **Un simulacro limpio no lo garantiza todo.** En `--check`, el módulo de
  Compose no puede saber que el `.env` va a cambiar, porque en simulacro
  el archivo no se escribe. La ejecución real recreó los contenedores.
  Lección: antes de la ejecución real, comprobar que los `.env` generados
  coinciden con los existentes y hacer una copia de la base de datos
  (`mariadb-dump`).
- **Nginx no arrancaba tras reiniciar.** Con el DNS interno, Nginx fallaba
  al arrancar con `host not found in upstream "vm2-app"`: resuelve el
  nombre del servidor solo al cargar la configuración, y en el arranque la
  red y el DNS aún no estaban listos. Se resolvió con un *drop-in* de
  systemd, que ahora instala el rol `nginx` (ver `docs/vm1-proxy.md`).
- **`prometheus.yml` con la configuración de ejemplo de Debian.** El
  simulacro del rol `prometheus` mostró que el archivo seguía teniendo la
  cabecera de ejemplo (`monitor: 'example'` y un Alertmanager inexistente
  en `localhost:9093`). Se revisó que no se usaba y la plantilla la
  elimina.
- **Permisos del archivo de Netplan.** El archivo de VM1 tenía permisos
  `644`; Netplan avisa de que deben ser `600` porque puede contener
  claves de red. El rol `dns_client` los corrige.
- **Las descargas no se pueden comprobar en simulacro.** En `--check`,
  `get_url` no descarga el archivo, así que no puede compararlo con el que
  hay y marca `changed` por precaución (repositorio de Docker). En la
  ejecución real lo compara y da `ok`. Con el dashboard de Grafana fue
  peor: para comprobarlo hace una petición `HEAD`, que la API de
  grafana.com rechaza con `405 Method Not Allowed`. Como esa descarga solo
  ocurre la primera vez (`force: false`), se salta en simulacro con
  `when: not ansible_check_mode`.
- **`site.yml` encontró un arreglo que no estaba aplicado.** Al lanzar
  todos los playbooks juntos en simulacro, `node_exporter` volvió a fallar
  en VM2 y VM4 con `Source ... not found`. El `check_mode: false` de la
  consulta de versión no había llegado a guardarse en el nodo de control,
  y no se notó porque en ejecución real el fallo no aparece. Es la ventaja
  de comprobar el laboratorio entero de vez en cuando.
- **SSH desde la máquina equivocada.** Ejecutar `ssh vm2-app` desde otra
  VM pedía contraseña, porque la clave SSH solo está en `ws`. Todo se
  ejecuta desde el nodo de control.

## Pendiente

- [ ] Zona horaria común en todas las VMs (VM3 tiene `-05:00`) desde
      `base.yml`.
- [ ] Probar `site.yml` contra máquinas recién instaladas.
- [ ] Despliegue en Azure con Terraform.
