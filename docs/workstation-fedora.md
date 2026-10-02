# Fedora - Puesto de trabajo (Control Node)

## Función

Máquina desde la que se administra todo el laboratorio: acceso por SSH a
VM1, VM2 y VM3, y punto desde donde se ejecutarán Ansible y Terraform. No
forma parte de la infraestructura web en sí, sino que es el "puesto de
sysadmin" del proyecto.

## Especificaciones

| | |
|---|---|
| Distro | Fedora Workstation |
| vCPU | 4 (1 procesador × 4 núcleos) |
| RAM | 8 GB |
| Disco | 60 GB |
| Hostname | ws |

## Red

| Adaptador | Tipo | IP |
|---|---|---|
| 1 | NAT | asignada por DHCP (salida a internet) |
| 2 | Host-only (VMnet1) | 192.168.56.10/24 |

Configurada con NetworkManager:
```bash
sudo nmcli connection add type ethernet ifname ens37 con-name host-only \
  ipv4.method manual ipv4.addresses 192.168.56.10/24
sudo nmcli connection up host-only
```

Resolución de nombres de las demás máquinas añadida en `/etc/hosts`:
```
192.168.56.10 ws
192.168.56.11 vm1-proxy
192.168.56.12 vm2-app
192.168.56.13 vm3-mon
```

## Software instalado

- `git`
- `vim`
- `curl`, `wget`
- `htop`
- `tmux`
- `bash-completion`
- `ansible`
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
9. Copiada la clave SSH pública a VM1, VM2 y VM3:
   ```bash
   ssh-copy-id usuario@vm1-proxy
   ssh-copy-id usuario@vm2-app
   ssh-copy-id usuario@vm3-mon
   ```

## Pendiente

- [ ] Instalar Terraform.
- [ ] Instalar Docker CLI y Azure CLI.
- [ ] Instalar VS Code (con la extensión Remote-SSH).
- [ ] Escribir los primeros playbooks de Ansible.

## Problemas encontrados

## Problemas encontrados

...
