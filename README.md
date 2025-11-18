# Plan de Resiliencia
Para este proyecto se usara 4 maquinas:
la direccion ip interna que usaremos sera 192.168.50.x


# 🏗️ Fase -1: Arquitectura de Hierro

## 🖥️ Especificaciones de Hardware
- **Host Global**: mínimo 8 GB RAM libres.  
- **Distribución de VMs:**

| VM | Hostname | RAM | CPU | Disco OS | Disco Datos | Red |
|----|----------|-----|-----|----------|-------------|-----|
| VM1 | `minio-vault` | 2048 MB | 2 | 25 GB | 20 GB (Backups) | NAT + Interna |
| VM2 | `app-node` | 1024 MB | 1 | 25 GB | - | Interna |
| VM3 | `db-node` | 1536 MB | 1 | 25 GB | 10 GB (LVM) | Interna |
| VM4 | `drp-control` | 1024 MB | 1 | 25 GB | - | Interna |

**Notas clave:**
- VM1 = Bastion Host (único acceso a internet).  
- VM3 = disco extra obligatorio para snapshots LVM.  
- Red Interna = aislada, solo VM1 conecta al exterior.

## En todas las máquinas

```bash
sudo apt update && sudo apt upgrade -y
```
> Para actualizar todos los paquetes
```bash
sudo apt install net-tools curl git htop nano -y
```
> Sirve para instalar herramientas basicas que vamos a usar
---
## Para configurar las ip staticas
### VM1 - minio-vault
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
>Esta es la configuracion de red
dentro de la configuracion de red
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.50.10/24
```
aplicar cambios
```bash
sudo netplan apply
```
Para activar reenvío de paquetes en el kernel
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
Instalar iptables-persistent para guardar las reglas
```bash
sudo apt install iptables-persistent -y
```
Es la regla NAT para que las demas maquinas tengan acceso a internet
```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo netfilter-persistent save
```
### VM2-app-node
abrir la configuracion de red
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
alli dentro
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.50.20/24
      routes:
        - to: default
          via: 192.168.50.10
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
aplicar cambios
```bash
sudo netplan apply
```
### VM3-db-node
abrir la configuracion de red
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
alli dentro
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.50.30/24
      routes:
        - to: default
          via: 192.168.50.10
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
aplicar cambios
```bash
sudo netplan apply
```
### VM4-drp-control
abrir la configuracion de red
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
alli dentro
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.50.40/24
      routes:
        - to: default
          via: 192.168.50.10
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
aplicar cambios
```bash
sudo netplan apply
```
