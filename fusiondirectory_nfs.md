# Manual de Operaciones: Integración de FusionDirectory, OpenLDAP y Servidor NFS

* **Asignatura:** Administración de Sistemas Operativos
* **Sistema Operativo:** Debian (Trixie/Bookworm)
* **Servicios:** Apache2, PHP, OpenLDAP (`slapd`), SSSD/NSS, NFS Kernel Server

---

## Resolución de Incidencias Técnicas (Troubleshooting)

### Error: `invalid OpenPGP data found`

Durante la configuración de repositorios externos PGP/GPG, si se intenta descargar la clave firmada directamente con un fichero formateado en ASCII o un archivo HTML/comprimido y pasarlo por `gpg --dearmor`, el gestor de paquetes `apt` lanza el error `invalid OpenPGP data found`.

**Solución / Contingencia aplicada:**

1. Se purga cualquier clave o lista corrupta de la configuración de APT:

   ```bash
   sudo rm -f /etc/apt/sources.list.d/fusiondirectory*
   sudo rm -f /etc/apt/keyrings/fusiondirectory.gpg
   ```

2. Para garantizar la continuidad del proyecto y la estabilidad de la infraestructura sin bloquear la actividad, se mantiene la pila web (Apache/PHP) y la integración de usuarios utilizando los repositorios oficiales de Debian.

---

## Módulo A: Gestión e Instalación del Entorno Web y de Directorio

### 1. Instalación de la Pila Web y Dependencias

Instalación de los paquetes base y módulos de integración LDAP desde los repositorios oficiales:

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-ldap php-mbstring php-xml php-curl php-intl -y
```

### 2. Validación del Servicio OpenLDAP y Esquemas POSIX

Verificación de que el servicio OpenLDAP local se encuentra en ejecución y que el sistema operativo resuelve correctamente los usuarios y grupos mediante SSSD/NSS:

```bash
sudo systemctl status slapd
getent passwd
getent group
```

> ![captura1](/img/captura1.png)
> ![captura2_1](/img/captura2_1.png)
> ![captura2_2](/img/captura2_2.png)

---

## Módulo B: Despliegue, Configuración e Integración del Servidor NFS

### 1. Instalación del Servidor y Cliente NFS

Instalación de los paquetes de kernel NFS y herramientas de administración:

```bash
sudo apt install nfs-kernel-server nfs-common -y
```

### 2. Creación del Directorio Compartido y Permisos POSIX

Creación de la carpeta de trabajo y vinculación de la propiedad al grupo POSIX gestionado en OpenLDAP (`desarrollo`), estableciendo permisos restringidos (`770`):

```bash
sudo mkdir -p /srv/nfs/desarrollo
sudo chown root:desarrollo /srv/nfs/desarrollo
sudo chmod 770 /srv/nfs/desarrollo
```

### 3. Configuración de Exportaciones (`/etc/exports`)

Edición del fichero de declaración de recursos compartidos para la red:

```bash
sudo nano /etc/exports
```

Añadir la directiva de exportación al final del fichero:

```text
/srv/nfs/desarrollo  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```

### 4. Aplicación y Reinicio del Servicio NFS

Actualización de la tabla de exportación del núcleo y reinicio del servicio:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

> ![captura3](/img/captura3.png)

---

## Módulo C: Verificación y Pruebas de Segregación de Acceso

### 1. Montaje Local del Recurso

Montaje de prueba en el punto `/mnt` para validar las políticas de acceso:

```bash
sudo mount -t nfs 127.0.0.1:/srv/nfs/desarrollo /mnt
```

### 2. Validación con Usuario Autorizado (Miembro del Grupo LDAP)

Se inicia sesión como el usuario de OpenLDAP perteneciente al grupo `desarrollo` y se intenta escribir en el directorio:

```bash
su - <usuario_autorizado>
touch /mnt/test_autorizado.txt
ls -l /mnt/test_autorizado.txt
```

* **Resultado esperado:** El fichero se crea correctamente ya que el usuario pertenece al grupo POSIX con permisos de escritura.

### 3. Validación con Usuario NO Autorizado (Fuera del Grupo)

Cambio de usuario a un usuario del sistema o de LDAP ajeno al grupo `desarrollo`:

```bash
su - <usuario_no_autorizado>
touch /mnt/test_prohibido.txt
```

* **Resultado esperado:** El sistema deniega la acción devolviendo el error `Permission denied`.

> ![captura4](/img/captura4.png)
