# Servidor GlusterFS con Alta Disponibilidad y Balanceo de Carga

Este proyecto describe la instalación y configuración de un clúster GlusterFS con alta disponibilidad y balanceo de carga utilizando **CTDB** y **Samba**, implementado en máquinas virtuales con **Ubuntu Server 24.04**.

## Requisitos

- VirtualBox (o entorno de virtualización)
- 3 máquinas virtuales:
  - `nodo1` y `nodo2` como servidores GlusterFS
  - `nodo3` como cliente
- Ubuntu Server 24.04
- Acceso root o sudo

---

## 1. Preparación de los nodos

1. Crear dos máquinas virtuales (nodo1 y nodo2) con red en modo *Adaptador puente*.
2. Instalar Ubuntu Server 24.04 en ambos nodos.
3. Actualizar el sistema:
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

## 2. Instalación de GlusterFS, CTDB y Samba

```bash
sudo add-apt-repository ppa:gluster/glusterfs-11 -y
sudo apt install glusterfs-server ctdb samba -y
sudo systemctl start glusterd
sudo systemctl enable glusterd
```

## 3. Configuración del almacenamiento

1. Añadir un segundo disco (`/dev/sdb`) y crear partición:
   ```bash
   sudo fdisk /dev/sdb
   # Opciones: n → p → 1 → Enter → Enter → w
   sudo mkfs.xfs /dev/sdb1
   ```

2. Montar y hacer persistente:
   ```bash
   sudo mkdir /glusterdata
   sudo mount /dev/sdb1 /glusterdata
   echo '/dev/sdb1 /glusterdata xfs defaults 0 0' | sudo tee -a /etc/fstab
   sudo mount -a
   ```
      Para comprobar que se ha montando correctamente:
   ```bash
   df -h
   ```
     Debe devolver algo como:

      ![Image](https://github.com/user-attachments/assets/feb1a892-a0d1-4855-9ec5-91f1b0bedb02)

      Donde podemos ver que /dev/sdb1 está montado en /glusterdata

4. Añadir IPs al archivo `/etc/hosts`.
  En este caso
  - `nodo1` : 192.168.2.104
  - `nodo2` : 192.168.2.253

## 4. Configuración de GlusterFS

1. Establecer relación de confianza:
   ```bash
   gluster peer probe nodo2
   ```

2. Crear bricks y volumen:
   ```bash
   sudo mkdir /glusterdata/vol1
   gluster volume create vol1 replica 2 nodo1:/glusterdata/vol1 nodo2:/glusterdata/vol1
   gluster volume start vol1
   ```

3. Montar el volumen:
   ```bash
   sudo mkdir /mnt/glustervol
   echo 'localhost:/vol1 /mnt/glustervol glusterfs defaults,_netdev 0 0' | sudo tee -a /etc/fstab
   sudo mount -a
   ```

---

## 5. Configuración de CTDB

Crear directorio de bloqueo:
```bash
sudo mkdir /mnt/glustervol/lock
```

Configurar archivos:

- `/mnt/glustervol/lock/ctdb`
  ```
  CTDB_RECOVERY_LOCK=/mnt/glustervol/lock/lockfile
  CTDB_PUBLIC_ADDRESSES=/etc/ctdb/public_addresses
  CTDB_NODES=/etc/ctdb/nodes
  CTDB_MANAGES_SAMBA=yes
  ```

- `/mnt/glustervol/lock/nodes` y `/etc/ctdb/nodes`
  ```
  192.168.2.104
  192.168.2.253
  ```

- `/etc/ctdb/public_addresses`
  ```
  192.168.2.200/24 enp0s3
  192.168.2.201/24 enp0s3
  ```

- `/etc/default/ctdb`
  ```
  CTDB_RECOVERY_LOCK=/mnt/glustervol/lock/lockfile
  ```

---

## 6. Configuración de Samba

Editar `/etc/samba/smb.conf`:

```ini
[global]
   clustering = yes
   idmap backend = tdb2
   private dir = /mnt/glustervol/lock
   netbios name = nodoX  # Diferente en cada nodo
   security = user
```
Al final del archivo añadir:
```ini
[glustertest]
   path = /mnt/glustervol
   read only = no
   guest ok = yes
```

---

## 7. Activar CTDB y comprobar estado

```bash
sudo systemctl disable smbd nmbd
sudo systemctl enable ctdb
sudo systemctl restart ctdb

ctdb status
ctdb ip
```
Ambos nodos tienen que estar marcados como OK:

![Image](https://github.com/user-attachments/assets/46e34521-706f-45f2-a623-28e85d7bf334)

Veremos las IPs virtuales y al lado un 0 o un 1; el 0 significa que es la ip está activa en este nodo, el 1 significa que está activa en el otro nodo. 

![Image](https://github.com/user-attachments/assets/c4641b64-bb04-4189-ae85-199a14ba3058)

---

## 8. Cliente GlusterFS (nodo3)



1. Crear una máquina virtual (nodo3) con red en modo *Adaptador puente*.
2. Instalar Ubuntu Server 24.04.
3. Actualizar el sistema:
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

4. Instalar glusterfs:

```bash
sudo apt install glusterfs-server -y
sudo systemctl start glusterd
sudo systemctl enable glusterd
```

5. Añadir un segundo disco (`/dev/sdb`) y crear partición:
   ```bash
   sudo fdisk /dev/sdb
   # Opciones: n → p → 1 → Enter → Enter → w
   sudo mkfs.xfs /dev/sdb1
   ```

6. Montar y hacer persistente:
   ```bash
   sudo mkdir /glusterdata
   sudo mount /dev/sdb1 /glusterdata
   echo '/dev/sdb1 /glusterdata xfs defaults 0 0' | sudo tee -a /etc/fstab
   sudo mount -a
   ```
      Para comprobar que se ha montando correctamente:
   ```bash
   df -h
   ```
     Debe devolver algo como:

      ![Image](https://github.com/user-attachments/assets/feb1a892-a0d1-4855-9ec5-91f1b0bedb02)

      Donde podemos ver que /dev/sdb1 está montado en /glusterdata

7. Añadir IP al archivo `/etc/hosts` de nodo1 y nodo2.
  En este caso
  - `nodo3` : 192.168.18.95

8. Establecer relación de confianza:
   En el nodo1
   ```bash
   gluster peer probe nodo3
   ```

9. Crear bricks y volumen:
   ```bash
   sudo mkdir /glusterdata/vol1
   gluster volume add-brick vol1 replica 3 nodo3:/glusterdata/vol1
   ```

10. Comprobar que se ha montado correctamente:
   ```bash
   gluster volume info
   gluster volume status
   ```

---

## 9. Verificar Alta Disponibilidad

1. Apagar uno de los nodos.
2. Desde nodo3:
   ```bash
   cat /mnt/glusterfs/test.txt
   nano /mnt/glusterfs/apagado.txt
   # Escribir: "nodo2 apagado"
   ```
3. Encender nodo apagado y comprobar replicación.

---
