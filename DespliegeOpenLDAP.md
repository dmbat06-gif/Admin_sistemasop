# Despliegue de Servidor OpenLDAP en Debian

Este repositorio contiene la guía paso a paso y la documentación técnica para la implementación de un servicio de directorio **OpenLDAP** sobre una máquina virtual con **Debian**, cumpliendo con los parámetros de red estática, FQDN y poblamiento inicial del directorio.

---

## Parámetros de la Práctica
* **Sistema Operativo:** Debian (VirtualBox)
* **Dirección IP:** `20.0.0.5/24`
* **FQDN:** `ldapserver.planetafp.local`
* **Dominio Base (DN):** `dc=planetafp,dc=local`

---

## Índice del Manual de Despliegue

1. [Configuración del Hostname y FQDN](#1-configuración-del-hostname-y-fqdn)
2. [Configuración de la Red Estática (Netplan)](#2-configuración-de-la-red-estática-netplan)
3. [Resolución de Nombres Locales (`/etc/hosts`)](#3-resolución-de-nombres-locales-etchosts)
4. [Instalación y Reconfiguración de OpenLDAP](#4-instalación-y-reconfiguración-de-openldap)
5. [Poblado del Directorio e Inserción de Datos](#5-poblado-del-directoriodo-e-inserción-de-datos)
6. [Evidencias y Capturas de Comprobación](#6-evidencias-y-capturas-de-comprobación)

---

### 1. Configuración del Hostname y FQDN
Establecemos el nombre de host de forma permanente en el sistema para que coincida con el dominio requerido:
```bash
sudo hostnamectl set-hostname ldapserver.planetafp.local
```

### 2. Configuración de la Red Estática (Netplan)
Configuramos la interfaz de red con una IP estática (`20.0.0.5/24`):
1. Creamos la ruta de Netplan si no existe y abrimos el archivo de configuración:
   ```bash
   sudo mkdir -p /etc/netplan
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
2. Añadimos el siguiente bloque (reemplazando `enp0s3` por el nombre de tu interfaz local):
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s3:
         dhcp4: no
         addresses:
           - 20.0.0.5/24
   ```
3. Otorgamos los permisos restrictivos necesarios y aplicamos los cambios:
   ```bash
   sudo chmod 600 /etc/netplan/01-netcfg.yaml
   sudo netplan apply
   ```

### 3. Resolución de Nombres Locales (`/etc/hosts`)
Actualizamos el archivo de hosts local para asociar la dirección IP estática y el bucle local con el FQDN del servidor:
```bash
sudo nano /etc/hosts
```
Añadiendo las siguientes líneas al final:
```text
127.0.0.1       localhost
127.0.1.1       ldapserver.planetafp.local    ldapserver
20.0.0.5        ldapserver.planetafp.local    ldapserver
```

### 4. Instalación y Reconfiguración de OpenLDAP
Actualizamos los repositorios e instalamos el servidor junto con sus utilidades de cliente:
```bash
sudo apt update
sudo apt install -y slapd ldap-utils netplan.io
```

Lanzamos el asistente interactivo de configuración para definir el árbol DIT base:
```bash
sudo dpkg-reconfigure slapd
```
* **Parámetros aplicados:**
  * ¿Omitir configuración? **No**
  * Dominio DNS: `planetafp.local`
  * Nombre de la organización: `planetafp`
  * Contraseña de administrador: *(Definida por el usuario)*

### 5. Poblado del Directorio e Inserción de Datos
Creamos un fichero en formato LDIF (`estructura_inicial.ldif`) para definir una unidad organizativa (`ou=Usuarios`) y un usuario de prueba (`uid=jperez`):

```ldif
dn: ou=Usuarios,dc=planetafp,dc=local
objectClass: organizationalUnit
ou: Usuarios

dn: uid=jperez,ou=Usuarios,dc=planetafp,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
cn: Juan Perez
sn: Perez
uid: jperez
uidNumber: 2001
gidNumber: 2001
homeDirectory: /home/jperez
loginShell: /bin/bash
userPassword: password123
mail: jperez@planetafp.local
```

Importamos los datos al servidor utilizando la cuenta de administrador:
```bash
sudo ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -f estructura_inicial.ldif
```

---

## 6. Evidencias y Capturas de Comprobación

### 6.1. Verificación de la Interfaz de Red y Dirección IP Estática
Comprobación mediante el comando `ip a` de que la máquina opera bajo la IP asignada `20.0.0.5/24`:
```bash
ip a
```
![Comprobación de IP](./images/ip_a.jpg)

### 6.2. Verificación del Hostname y FQDN
Comprobación del nombre de host estático configurado en el sistema operativo:
```bash
hostnamectl
```
![Comprobación de Hostname](./images/images/hostnamectl.jpg)

### 6.3. Búsqueda y Validación del Usuario en el Directorio
Consulta LDAP para comprobar que el usuario `jperez` se encuentra registrado y operativo en el DIT:
```bash
ldapsearch -x -b "dc=planetafp,dc=local" -s sub "(uid=jperez)"
```
![Consulta LDAP del Usuario](./images/images/ulta-s_sub_uid=jperez.jpg)
