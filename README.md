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
- para ver los discos en una vm el comando es:
  ```bash
  lsblk
  ```

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
> # PARA PROBAR QUE TODO ESTA BIEN HASTA ESTE PUNTO CADA MAQUINA DEBE PODER HACER PING A GOOGLE.COM
## Generar claves SSH
### VM4-drp-control
Esto sirve para crear un par de llaves criptográficas una publica y otra privada
```bash
ssh-keygen -t ed25519 -C "ansible-control" -f ~/.ssh/id_ed25519 -N ""
```
Se debe copiar las llaves publicas a los demas servidores para poder entrar a ellas sin usar contrasenia, en este ejemplo todas mis maquinas tienen el usuario "jhoel", pero si se tiene otro usuario se las debe poner el nombre del usuario en vez de jhoel segun la maquina a la que corresponda

Copiar llave a VM1 (MinIO Vault)
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub jhoel@192.168.50.10
```
Copiar llave a VM2 (App Node)
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub jhoel@192.168.50.20
```
Copiar llave a VM3 (DB Node)
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub jhoel@192.168.50.30
```
> Para probar que funciona, el vm4 debe intentar acceder a cualquier otra maquina
> si le pide contrasenia entonces algo fallo
> si es que no le pide la contrasenia ni le deja entrar algo fallo
> si es que no le pide la contrasenia pero si deja entrar entonces esta bien

## Instalacion de otras Herramientas
### VM3-db-node
Para verificar el los discos propuestos si existes:
```bash
lsblk
```
> Debe aparecerte el disco de 10 gb, comunmente aparece como sdb o otro
Para instalar un gestor de volumenes logicos (lvm2)
```bash
sudo apt update && sudo apt install lvm2 -y
```
## VM1 - minio-vault
Usaremos Docker para levantar MinIO, asi que instalamos docker
```bash
sudo apt update && sudo apt install docker.io -y
```
para agregar al usuario actual al grupo de usuarios de Docker en el sistema
Es decir darle permiso a nuestro usario
```bash
sudo usermod -aG docker $USER
```
## VM4-drp-control
Instalar ansible para orquestar el DRP es decir automatizar
```bash
sudo apt update && sudo apt install ansible -y
```
## Despliegue de MinIO
## VM1-minio-vault
crearemos esta carpeta para que minio guarde ahi los objetos o datos
```bash
sudo mkdir -p /mnt/data
```
para darle permisos a minio sobre esa carpeta
```bash
sudo chown -R 1001:1001 /mnt/data
```
