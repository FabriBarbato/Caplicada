**Tutorial Completo: Configuración de SSH y LAMP en Debian VM con Troubleshooting**

Este manual reúne paso a paso todo lo realizado para:

1. importar la VM,
2. recuperar contraseña root,
3. habilitar acceso SSH por clave,
4. instalar y configurar Apache + PHP + MariaDB,
5. configurar red estática,
6. añadir, particionar y montar un segundo disco,
7. montar automáticamente en `/etc/fstab`,
8. mover el sitio web a un volumen dedicado,
9. configurar `/proc`,
10. crear script de backup y cron,
11. depurar y añadir extensión mysqli.

Incluye troubleshooting para cada problema encontrado.

---

## 1. Descarga y ensamblado de la VM

1. Descarga los archivos `TPVMCA.part01.rar` … `TPVMCA.part09.rar` desde Blackboard.
2. En Windows, con WinRAR o 7-Zip, haz clic derecho en **TPVMCA.part01.rar** → **Extraer aquí**. Esto genera `TPVMCA.ova`.

**Troubleshooting:**

* Error de volumen faltante: verifica que todos los `.rar` existen y no están renombrados.

---

## 2. Importar la VM en VirtualBox

1. Abre Oracle VirtualBox.
2. **Archivo › Importar aplicación virtualizada…**, selecciona `TPVMCA.ova`.
3. Acepta valores por defecto. Ajusta RAM (≥2 GB) y CPU (≥2 núcleos) si deseas.

**Troubleshooting:**

* Si sale “OVF version unknown”, actualiza VirtualBox.

---

## 3. Arranque en modo single-user y cambio de contraseña `root`

1. Inicia la VM. En GRUB selecciona **Debian GNU/Linux** y pulsa **e**.
2. En la línea que empieza por `linux /boot/vmlinuz… ro quiet`, añade al final: `init=/bin/bash`.
3. Pulsa **Ctrl+X** para arrancar en shell.
4. Monta `/` en lectura-escritura:

   ```bash
   mount -o remount,rw /
   ```
5. Cambia la contraseña de root:

   ```bash
   passwd root
   # escribe “palermo” (o la que prefieras)
   ```
6. Continúa el arranque normal:

   ```bash
   exec /sbin/init
   ```

**Troubleshooting:**

* Si arranca sin cambios, revisa que `init=/bin/bash` esté en la misma línea del kernel.

---

## 4. Configuración de red y acceso a Internet

1. Obtén la interfaz y solicita DHCP si no hay IP:

   ```bash
   ip addr show
   dhclient enp0s3
   ```
2. Comprueba conectividad:

   ```bash
   ping -c 4 8.8.8.8
   ping -c 4 deb.debian.org
   ```
3. Si falla DNS, edita `/etc/resolv.conf`:

   ```text
   nameserver 8.8.8.8
   ```

---

## 5. Configuración de repositorios APT

1. Crea `/etc/apt/sources.list`:

   ```bash
   cat << 'EOF' > /etc/apt/sources.list
   deb http://deb.debian.org/debian bullseye main contrib non-free
   deb http://deb.debian.org/debian-security bullseye-security main contrib non-free
   deb http://deb.debian.org/debian bullseye-updates main contrib non-free
   EOF
   ```
2. `apt update`

**Troubleshooting:**

* Si no puedes escribir `<`, usa `echo` con redirección `>` y `>>`.

---

## 6. Instalación de OpenSSH Server y configuración de SSH por clave

1. Instala SSH:

   ```bash
   apt install -y openssh-server
   systemctl status ssh
   ```
2. Prepara `/root/.ssh`:

   ```bash
   mkdir -p /root/.ssh
   chmod 700 /root/.ssh
   ```
3. Habilita password auth temporal y root login:

   ```bash
   sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config
   sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/'      /etc/ssh/sshd_config
   systemctl restart ssh
   ```
4. Configura port‑forwarding NAT en VirtualBox:

   * NAT › Avanzado › Reenvío de puertos: `127.0.0.1:2222 → 22`
5. Desde Windows PowerShell copia la clave pública:

   ```powershell
   cd "C:\Users\Fabri\Documents\…\claves"
   scp -i .\clave_privada.txt -P 2222 .\clave_publica.pub root@127.0.0.1:/root/
   ```
6. En la VM:

   ```bash
   cat /root/clave_publica.pub >> /root/.ssh/authorized_keys
   chmod 600 /root/.ssh/authorized_keys
   rm /root/clave_publica.pub
   ```
7. Deshabilita password auth y root login por contraseña:

   ```bash
   sed -i 's/^PasswordAuthentication .*/PasswordAuthentication no/'             /etc/ssh/sshd_config
   sed -i 's/^PermitRootLogin .*/PermitRootLogin prohibit-password/'           /etc/ssh/sshd_config
   systemctl restart ssh
   ```
8. Prueba SSH desde Windows:

   ```powershell
   ssh -i .\clave_privada.txt -p 2222 root@127.0.0.1
   ```

**Troubleshooting:**

* Errores de SCP: verifica passphrase, password root, port‑forward.
* Si no puedes pegar en VirtualBox, copia con SCP o carpeta compartida.

---

## 7. Instalación de Apache y PHP ≥ 7.3

1. `apt install -y apache2 php libapache2-mod-php`
2. `systemctl status apache2` debe estar **active (running)**
3. Monta carpeta compartida Blackboard si la usas:

   ```bash
   mkdir -p /mnt/blackboard
   mount -t vboxsf Blackboard /mnt/blackboard
   ```
4. Copia tu web:

   ```bash
   cp /mnt/blackboard/index.php /var/www/html/
   cp /mnt/blackboard/logo.png   /var/www/html/
   chown www-data:www-data /var/www/html/*
   chmod 644               /var/www/html/*
   ```

**Troubleshooting:**

* Si index.php sale en blanco, revisa logs en `/var/log/apache2/error.log`.

---

## 8. Instalación y configuración de MariaDB

1. `apt install -y mariadb-server mariadb-client`
2. `systemctl enable --now mariadb` (o `mysql` según unidad)
3. `systemctl status mariadb` debe mostrar **active (running)**
4. Importa tu script:

   ```bash
   mariadb -u root < /root/db.sql
   ```
5. Verifica:

   ```bash
   mariadb -u root -e "SHOW DATABASES;"
   ```

**Troubleshooting:**

* Si da socket error, arranca el servicio correcto.

---

## 9. Configuración de IP estática

1. `vi /etc/network/interfaces`
2. Añade:

   ```text
   auto enp0s3
   iface enp0s3 inet static
     address 10.0.2.15
     netmask 255.255.255.0
     gateway 10.0.2.2
     dns-nameservers 8.8.8.8 8.8.4.4
   ```
3. `systemctl restart networking`
4. Verifica con `ip addr show enp0s3` y `ping`.

---

## 10. Añadir un segundo disco de 10 GB en VirtualBox

**Host Windows:**

1. Apaga VM.
2. Configuración → Almacenamiento → Controlador SATA → Añadir disco → Crear VDI 10 GB.
3. Inicia VM.

**VM Debian:**
4\. `lsblk` → detecta `/dev/sdc` de 10 GB.
5\. `fdisk /dev/sdc`:

* `n` → `p` → `1` → **Enter** → `+3G` → **Enter**
* `n` → `p` → `2` → **Enter** → `+6G` → **Enter**
* `w` → **Enter**

6. `lsblk /dev/sdc` debe mostrar `sdc1` (3 G) y `sdc2` (6 G).

---

## 11. Formatear y montar particiones

```bash
mkfs.ext4 /dev/sdc1
mkfs.ext4 /dev/sdc2
mkdir -p /www_dir /backup_dir
mount /dev/sdc1 /www_dir
mount /dev/sdc2 /backup_dir
df -h | grep -E '/www_dir|/backup_dir'
```

---

## 12. Automontaje en `/etc/fstab`

1. Obtén UUIDs:

   ```bash
   blkid /dev/sdc1 /dev/sdc2
   ```
2. Edita `/etc/fstab` y añade:

   ```text
   UUID=8f2880d-68a0-4db7-bd8c-19733c5ea5a7  /www_dir    ext4  defaults  0 2
   UUID=f675a643-5101-4280-861d-97061f74483e  /backup_dir ext4  defaults  0 2
   ```
3. Prueba:

   ```bash
   umount /www_dir /backup_dir
   mount -a
   df -h | grep dir
   ```

---

## 13. Reubicar sitio web a `/www_dir`

```bash
systemctl stop apache2
mv /var/www/html/* /www_dir/
chown -R www-data:www-data /www_dir
find /www_dir -type d -exec chmod 755 {} \;
find /www_dir -type f -exec chmod 644 {} \;
vi /etc/apache2/sites-available/000-default.conf
  DocumentRoot /www_dir
systemctl start apache2
```

**Troubleshooting:** si 403, añade en `/etc/apache2/apache2.conf`:

```apache
<Directory /www_dir>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

`systemctl reload apache2`

---

## 14. Chequeo de Apache sin GUI

**Dentro de la VM**:

```bash
curl -I http://localhost   # 200 OK
```

**Desde Windows** (port-forward 8080→80):

```powershell
curl http://127.0.0.1:8080 -UseBasicParsing
Test-NetConnection -ComputerName 127.0.0.1 -Port 8080
```

---

## 15. Prueba PHP y logs

1. Crea `/www_dir/test.php`:

   ```bash
   cat > /www_dir/test.php << 'EOF'
   <?php phpinfo();
   EOF
   systemctl reload apache2
   ```
2. Visita `http://127.0.0.1:8080/test.php` o `curl http://localhost/test.php`.
3. Si sale en blanco, revisa:

   ```bash
   apache2ctl -M | grep php
   vi /etc/apache2/mods-enabled/dir.conf   # DirectoryIndex index.php
   ```
4. Logs de errores:

   ```bash
   tail -n 20 /var/log/apache2/error.log
   ```

---

## 16. Habilitar la extensión MySQLi en PHP

```bash
apt install -y php-mysql
systemctl restart apache2
```

Prueba tu `index.php` y revisa logs si surge algo.

---

## 17. Tareas pendientes (próxima entrega)

* Crear enlace `/proc/partitions` en `/proc/particion`.
* Escribir script de backup completo en `/opt/scripts/backup_full.sh`.
* Programar cron jobs para respaldos.

*Fin del tutorial de hoy.*
