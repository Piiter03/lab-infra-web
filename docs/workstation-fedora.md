# ws - Puesto de trabajo (Control Node)

## Función

Máquina desde la que se administra todo el laboratorio: acceso por SSH a
las VMs de servidor (VM1 a VM5), y punto desde donde se ejecutarán Ansible y
Terraform. No forma parte de la infraestructura web en sí, sino que es el
"puesto de sysadmin" del proyecto.

## Especificaciones

| | |
|---|---|
| Distro | Fedora Workstation |
| vCPU | 4 (1 procesador × 4 núcleos) |
| RAM | 8 GB |
| Disco | 60 GB |
| Hostname | ws |

## Red

| Adaptador | Tipo | Interfaz | IP |
|---|---|---|---|
| 1 | NAT | `ens224` (conexión `Wired connection 1`) | asignada por DHCP (salida a internet) |
| 2 | Host-only (VMnet1) | `ens160` (conexión `host-only`) | 192.168.56.10/24 |

Configurada con NetworkManager:
```bash
sudo nmcli connection add type ethernet ifname ens160 con-name host-only \
  ipv4.method manual ipv4.addresses 192.168.56.10/24
sudo nmcli connection up host-only
```

DNS: usa el DNS interno de VM5 (ver `docs/vm5-dns.md`):
```bash
sudo nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
sudo nmcli connection modify host-only ipv4.dns 192.168.56.15
sudo nmcli connection up "Wired connection 1"
sudo nmcli connection up host-only
resolvectl status
```

Ya no hay entradas del laboratorio en `/etc/hosts`: todos los nombres
(`vm1-proxy`, `vm2-app`, `vm3-mon`, `vm4-db`, `vm5-dns`) se resuelven por
DNS.

## Software instalado

- `git`
- `vim`
- `curl`, `wget`
- `htop`
- `tmux`
- `bash-completion`
- `ansible`
- `bind-utils` (para `dig`)
- `open-vm-tools`, `open-vm-tools-desktop`
- *(pendiente: Terraform, Docker CLI, Azure CLI, VS Code)*

## Pasos realizados

1. Instalación de Fedora Workstation desde la ISO oficial.
2. Actualización del sistema:
   ```bash
   sudo dnf upgrade --refresh -y
   ```
3. Comprobación/instalación de las herramientas de integración con VMware:
   ```bash
   sudo dnf install -y open-vm-tools open-vm-tools-desktop
   sudo systemctl enable --now vmtoolsd
   ```
4. Cambio del nombre de host:
   ```bash
   sudo hostnamectl set-hostname ws
   ```
5. Instalación de herramientas básicas y Ansible:
   ```bash
   sudo dnf install -y git vim curl wget htop tmux bash-completion ansible
   ```
6. Generación de una clave SSH para acceder al resto de VMs sin contraseña:
   ```bash
   ssh-keygen -t ed25519 -C "ws"
   ```
7. Snapshot de la VM ya configurada (`fedora-base-limpia`) para poder volver
   a este punto si algo se rompe más adelante.
8. Añadido un segundo adaptador de red (Host-only, VMnet1) y configurada su
   IP fija con NetworkManager (ver sección **Red**).
9. Copiada la clave SSH pública a las VMs de servidor:
   ```bash
   ssh-copy-id usuario@vm1-proxy
   ssh-copy-id usuario@vm2-app
   ssh-copy-id usuario@vm3-mon
   ssh-copy-id usuario@vm4-db
   ssh-copy-id usuario@vm5-dns
   ```
10. Migración al DNS interno (VM5) y eliminación de las entradas del
    laboratorio de `/etc/hosts`.

## Pendiente

- [ ] Instalar Terraform.
- [ ] Instalar Docker CLI y Azure CLI.
- [ ] Instalar VS Code (con la extensión Remote-SSH).
- [ ] Escribir los primeros playbooks de Ansible.

## Problemas encontrados

- Con el DNS interno, `ws` depende de VM5 para resolver los nombres del
  laboratorio. Si VM5 está apagada, el acceso por nombre falla, aunque el
  acceso por IP sigue funcionando.
