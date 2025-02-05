# 📖 Manual: Montar un disco en Proxmox y compartirlo con LXC y VM usando Bind Mounts + VirtioFS

Este manual explica cómo montar un disco en **Proxmox**, compartirlo con **contenedores LXC** y hacer que una **máquina virtual (VM)** también lo vea mediante **VirtioFS**.

---

## 🔍 Paso 1: Identificar el disco en Proxmox
Ejecuta el siguiente comando para listar los discos disponibles:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

Ejemplo de salida:
```bash
NAME   SIZE  FSTYPE MOUNTPOINT
sda    1.8T  zfs    -
sdb    1.8T  zfs    -
sdc    256G  ext4   /
sdd    10T          -
```

El disco que queremos montar es `/dev/sdd`, ya que tiene 10TB y no tiene sistema de archivos asignado.

Si quieres más información sobre particiones:
```bash
fdisk -l /dev/sdd
```

---

## 📌 Paso 2: Crear punto de montaje en Proxmox
1. Crear el directorio donde montaremos el disco:
   ```bash
   mkdir -p /mnt/almacenamiento
   ```
2. Montar el disco manualmente para probar:
   ```bash
   mount /dev/sdd /mnt/almacenamiento
   ```
3. Para que se monte automáticamente al reiniciar, editar `/etc/fstab`:
   ```bash
   nano /etc/fstab
   ```
   Añadir esta línea al final:
   ```
   /dev/sdd  /mnt/almacenamiento  ext4  defaults  0  2
   ```
4. Aplicar cambios:
   ```bash
   mount -a
   ```

---

## 🔄 Paso 3: Compartir el disco con los contenedores LXC
1. **Identificar el ID del contenedor**:
   ```bash
   pct list
   ```
   Ejemplo de salida:
   ```bash
   CTID    Status    IP Address       
   100     running   192.168.2.31   
   ```
2. **Editar la configuración del contenedor**:
   ```bash
   nano /etc/pve/lxc/100.conf
   ```
3. **Añadir esta línea** para montar el disco dentro del contenedor:
   ```
   mp0: /mnt/almacenamiento,mp=/mnt/almacenamiento
   ```
4. **Reiniciar el contenedor**:
   ```bash
   pct restart 100
   ```

---

## 🖥 Paso 4: Compartir el disco con una máquina virtual (VM) usando VirtioFS
1. **Identificar el ID de la VM**:
   ```bash
   qm list
   ```
   Ejemplo de salida:
   ```bash
   VMID    Name        Status  
   200     UbuntuVM    running
   ```
2. **Editar la configuración de la VM**:
   ```bash
   nano /etc/pve/qemu-server/200.conf
   ```
3. **Añadir esta línea al final**:
   ```
   args: -device vhost-user-fs-pci,chardev=char9,tag=almacenamiento -chardev socket,id=char9,path=/mnt/almacenamiento
   ```
4. **Reiniciar la VM**:
   ```bash
   qm restart 200
   ```
5. **Montar el directorio dentro de la VM** (Linux):
   ```bash
   mkdir -p /mnt/almacenamiento
   mount -t virtiofs almacenamiento /mnt/almacenamiento
   ```
6. Para que el montaje sea permanente, añadir a `/etc/fstab` dentro de la VM:
   ```
   almacenamiento /mnt/almacenamiento virtiofs defaults 0 0
   ```

---

## ✅ ¡Listo!
Ahora, el disco está montado en Proxmox, accesible en los **contenedores LXC** y también en la **máquina virtual** mediante VirtioFS.

Si necesitas hacer cambios o ampliar la configuración, revisa los permisos y verifica los montajes con:
```bash
ls -lah /mnt/almacenamiento
