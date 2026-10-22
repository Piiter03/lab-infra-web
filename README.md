# Laboratorio de Infraestructura Web

Infraestructura web de 3 capas (presentación, aplicación, datos) más
servicios de infraestructura (monitorización y DNS interno), montada como
laboratorio de virtualización para practicar administración de sistemas
Linux y, más adelante, despliegue en la nube con Infraestructura como
Código.

> **⏸️ Proyecto en pausa (octubre de 2026)**
>
> La fase de automatización está terminada: todo el laboratorio se
> configura con Ansible y se aplica con un solo comando (`site.yml`), con
> credenciales cifradas en Ansible Vault.
>
> Lo dejo en pausa para preparar la certificación **AZ-104 (Microsoft
> Azure Administrator)**. Al retomarlo, la siguiente fase será desplegar
> esta misma arquitectura en Azure con Terraform, aplicando lo aprendido.

## Objetivo del proyecto

Ganar experiencia práctica en sistemas y cloud replicando un entorno
similar al de una empresa: distintas distribuciones Linux, servicios
reales, seguridad básica, aislamiento real entre capas, monitorización y
resolución de nombres interna, con el objetivo final de automatizarlo con
Ansible y desplegarlo en Azure con Terraform.

## Arquitectura

```mermaid
flowchart LR
    U([Usuario<br/>desde ws]) -->|HTTPS 443| P

    subgraph L1 [Capa de presentación]
        P[VM1 · vm1-proxy<br/>Nginx + HTTPS]
    end
    subgraph L2 [Capa de aplicación]
        A[VM2 · vm2-app<br/>WordPress en Docker]
    end
    subgraph L3 [Capa de datos]
        D[VM4 · vm4-db<br/>MariaDB en Docker]
    end

    P -->|HTTP 8080| A
    A -->|MySQL 3306| D

    subgraph INF [Infraestructura transversal]
        M[VM3 · vm3-mon<br/>Prometheus + Grafana]
        N[VM5 · vm5-dns<br/>dnsmasq<br/>DNS de todas las VMs]
    end

    P -. node_exporter .-> M
    A -. node_exporter .-> M
    D -. node_exporter .-> M
    N -. node_exporter .-> M

```

**Las 3 capas reales:**
- **Presentación** → VM1 (Nginx, proxy inverso con HTTPS)
- **Aplicación / lógica de negocio** → VM2 (WordPress en Docker)
- **Datos** → VM4 (MariaDB en Docker, aislada en su propia máquina)

**Infraestructura transversal** (no son capas de la arquitectura):
- **Observabilidad** → VM3 (Prometheus + Grafana)
- **Resolución de nombres** → VM5 (dnsmasq)

## Máquinas virtuales

| VM | Función | Distro | RAM | vCPU | Disco | IP (host-only) |
|---|---|---|---|---|---|---|
| ws | Puesto de trabajo (control node) | Fedora Workstation | 8 GB | 4 | 60 GB | 192.168.56.10 |
| VM1 | Proxy inverso (Nginx, HTTPS) | Ubuntu Server LTS | 2 GB | 1 | 20 GB | 192.168.56.11 |
| VM2 | Aplicación (WordPress) | Rocky Linux | 4 GB | 2 | 30 GB | 192.168.56.12 |
| VM3 | Monitorización (Prometheus + Grafana) | Debian | 4 GB | 2 | 40 GB | 192.168.56.13 |
| VM4 | Base de datos (MariaDB) | Rocky Linux | 4 GB | 2 | 30 GB | 192.168.56.14 |
| VM5 | DNS interno (dnsmasq) | Debian | 1 GB | 1 | 10 GB | 192.168.56.15 |

Virtualizado con **VMware Workstation Pro** sobre Windows. Todas las VMs
tienen un adaptador NAT (internet) y un adaptador Host-only (VMnet1, red
interna del laboratorio).

## Stack utilizado

- **Proxy:** Nginx, HTTPS con certificado autofirmado
- **Aplicación:** WordPress en contenedor Docker
- **Base de datos:** MariaDB en contenedor Docker, en VM dedicada
- **Monitorización:** Prometheus, node_exporter, Grafana
- **DNS interno:** dnsmasq
- **Seguridad:** SELinux (Rocky), firewalld/ufw
- **Red:** red interna host-only con IPs fijas y resolución por DNS interno
- **Automatización:** Ansible (todo el laboratorio: DNS, Nginx, Docker, aplicaciones con Vault, Prometheus y Grafana)
- **Próximamente:** Terraform (despliegue en Azure, tras la AZ-104)

## Estado del proyecto

- [x] Instalación y configuración base de las 6 máquinas
- [x] Red interna host-only con IPs fijas entre todas las máquinas
- [x] Acceso SSH sin contraseña desde `ws` a las VMs de servidor
- [x] Firewall activo en las VMs de servidor (ufw / firewalld)
- [x] WordPress desplegado en Docker en VM2
- [x] Base de datos separada en su propia VM (VM4), con aislamiento real
- [x] Proxy inverso en VM1 con HTTPS, funcionando de extremo a extremo
- [x] node_exporter en VM1, VM2, VM4 y VM5, monitorizadas desde VM3
- [x] Dashboard de Grafana con métricas de las máquinas
- [x] DNS interno (VM5) y migración de todos los clientes
- [x] Ansible: inventario, configuración base, servidor DNS y node_exporter en las 5 VMs
- [x] Ansible: Docker y despliegue de WordPress y MariaDB con credenciales en Vault
- [x] Ansible: cliente DNS (con un caso por distribución), Nginx y Prometheus como roles
- [x] Ansible: Grafana como código y un `site.yml` para aplicar el laboratorio completo
- [ ] Despliegue en Azure con Terraform *(en pausa hasta completar la AZ-104)*

## Automatización con Ansible

Toda la configuración del laboratorio está también como código en
[`ansible/`](ansible), y se aplica desde `ws`:

| Playbook | Máquinas | Qué hace |
|---|---|---|
| `base.yml` | todas | paquetes comunes |
| `dns.yml` | VM5 | servidor DNS (dnsmasq) |
| `dns_client.yml` | VM1-VM4 | uso del DNS interno con `nmcli`, Netplan o `dhcpcd` según la distro |
| `node_exporter.yml` | todas | métricas para Prometheus |
| `monitoring.yml` | VM3 | Prometheus (un job por máquina, generado desde el inventario) y Grafana (fuente de datos y dashboard como código) |
| `proxy.yml` | VM1 | Nginx como proxy inverso con HTTPS |
| `docker.yml` | VM2, VM4 | Docker CE y Compose |
| `apps.yml` | VM4, VM2 | MariaDB y WordPress, con las credenciales en Ansible Vault |
| `site.yml` | todas | todo lo anterior, en orden: el laboratorio completo con un comando |

```bash
cd ansible
ansible-playbook <playbook>.yml --check --diff -K --ask-vault-pass   # simulacro
ansible-playbook <playbook>.yml -K --ask-vault-pass                  # ejecución real
```

Todos los playbooks son idempotentes (una segunda ejecución da
`changed=0`), validan la configuración antes de aplicarla (`nginx -t`,
`promtool`, `dnsmasq --test`) y no guardan ninguna contraseña en texto
plano. Detalles, comandos y problemas en
[docs/ansible.md](docs/ansible.md).

## Documentación por máquina

- [ws - Puesto de trabajo](docs/workstation-fedora.md)
- [VM1 - Proxy Nginx](docs/vm1-proxy.md)
- [VM2 - Aplicación (WordPress)](docs/vm2-app.md)
- [VM3 - Monitorización](docs/vm3-monitorizacion.md)
- [VM4 - Base de datos](docs/vm4-db.md)
- [VM5 - DNS interno](docs/vm5-dns.md)
- [Automatización con Ansible](docs/ansible.md)

## Lo que he aprendido

- Configuración de red host-only y NAT en VMware para simular un entorno de servidores, incluyendo varios fallos de adaptador mal asignado (IP fija en la interfaz equivocada, segundo adaptador faltante, o apuntando a una red host-only distinta de la compartida por el resto de VMs). Además, el orden de los adaptadores y los nombres de interfaz cambian de una VM a otra, así que siempre hay que comprobar `ip a` antes de editar la red.
- Diferencias de administración entre familias de distros (`apt` en Debian/Ubuntu vs `dnf`/SELinux/firewalld en Rocky), y entre gestores de red (`nmcli`, Netplan, `dhcpcd`, `/etc/network/interfaces`).
- Diagnóstico de fallos de resolución y conectividad usando `ip a`, `ip route`, `nmcli`, `resolvectl` y `ping`, capa por capa (red → firewall → servicio).
- Despliegue de aplicaciones en contenedores Docker, incluyendo que **los contenedores no heredan el `/etc/hosts` del sistema anfitrión**. Primero se resolvió con `extra_hosts`, y después de forma definitiva con un DNS interno, cuyo resolvedor sí hereda el contenedor.
- Corrección de un diseño de arquitectura de 3 capas mal aislado (base de datos en la misma VM que la aplicación) tras recibir una observación técnica válida, separando la base de datos en una VM dedicada (VM4) con su propio aislamiento de red y firewall.
- Configuración de Nginx como proxy inverso con HTTPS, reenviando peticiones a un servicio en otra máquina.
- Gestión de zonas y `sources` en firewalld (un mismo origen no puede pertenecer a dos zonas a la vez).
- Despliegue de Prometheus y Grafana para monitorización multi-máquina, incluyendo la instalación manual de `node_exporter` por binario en distros que no lo traen en sus repositorios, y la depuración de un `prometheus.yml` inválido. De ahí salió el hábito de **validar antes de reiniciar** (`promtool check config`, `nginx -t`, `dnsmasq --test`, `netplan try`).
- Montaje de un DNS interno con dnsmasq y migración de clientes con cuatro métodos distintos. En Ubuntu, `systemd-resolved` **no reenvía los nombres de una sola etiqueta** a un DNS externo salvo que se active `ResolveUnicastSingleLabel=yes`.
- Automatización con Ansible de una infraestructura heterogénea: roles, plantillas Jinja2, `host_vars` para las diferencias entre máquinas, *handlers* que validan antes de recargar, simulacro (`--check --diff`) e idempotencia, y credenciales con **Ansible Vault**. Un fallo real enseñó que un simulacro limpio no lo garantiza todo (el módulo de Compose no ve un `.env` que aún no se ha escrito), y que un `.env` con variables desplazadas puede pasar desapercibido porque MariaDB solo lo lee al crear la base de datos.
- Dependencias de arranque: con el DNS interno, Nginx fallaba al iniciar (`host not found in upstream`) porque resuelve el destino solo al cargar la configuración. Se arregló con un *drop-in* de systemd.
- Valoración de dependencias de infraestructura: el DNS interno es un punto único de fallo, y se decidió separarlo de la monitorización para evitar una dependencia circular.

## Capturas

**Docker funcionando en VM2:**

![Docker ejecutando el contenedor hello-world](docs/img/docker-hello-world.png)

**WordPress accesible a través del proxy de VM1:**

![WordPress funcionando a través del proxy HTTPS](docs/img/wordpress-funcionando.png)

**DNS interno:**

![dnsmasq en ejecución en VM5](docs/img/vm5-dnsmasq-status.png)
![Consultas dig desde ws a vm5-dns](docs/img/vm5-dig-desde-ws.png)
![Contenedor de WordPress resolviendo vm4-db sin extra_hosts](docs/img/vm2-docker-dns.png)

**Monitorización con Prometheus y Grafana:**

![Targets de Prometheus con las 6 máquinas en estado UP](docs/img/vm3-prometheus-targets.png)
![Dashboard de Grafana con métricas de node_exporter](docs/img/vm3-grafana-dashboard.png)

## Autor

Pedro Morales — en proceso de formación en sistemas y cloud (ASIR, AZ-900)

[LinkedIn](https://www.linkedin.com/in/pedro-morales03/) · [GitHub](https://github.com/Piiter03)
