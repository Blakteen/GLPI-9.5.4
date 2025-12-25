# Guía Profesional de Instalación: GLPI 9.5.4 en Ubuntu Server

## Introducción

Esta guía ha sido diseñada para la instalación de **GLPI 9.5.4** en un servidor Ubuntu LTS. Se incluyen explicaciones detalladas de cada paso, recomendaciones de seguridad y optimización del sistema, siguiendo las **mejores prácticas** de un entorno de producción.

---

## 1. Contexto y Requisitos Previos

Antes de iniciar el proceso, es fundamental confirmar que el entorno cumple con los siguientes requisitos técnicos:

| Componente | Versión Mínima | Observaciones y Recomendaciones |
| :--- | :--- | :--- |
| **Ubuntu LTS** | 20.04 o 22.04 | Para Ubuntu 22.04, se recomienda encarecidamente utilizar **PHP 8.0** para asegurar la compatibilidad. |
| **PHP** | 7.2 – 8.0 | PHP 8.0 es la versión óptima (*sweet spot*) para GLPI 9.5.4. |
| **MariaDB** | 10.3+ | Se prefiere MariaDB sobre MySQL 5.7 por su estabilidad y rendimiento mejorados. |
| **RAM** | 2 GB | 1 GB es suficiente para entornos pequeños con menos de 100 equipos. |
| **Almacenamiento** | 20 GB | 10 GB para el sistema operativo y la base de datos, y el resto para archivos adjuntos y logs. |

> **⚠️ Nota de Seguridad:** Todos los comandos deben ejecutarse con un usuario que posea privilegios `sudo`. **Nunca utilice el usuario `root` directamente** para la instalación, a menos que sea estrictamente necesario.

---

## 2. Preparación del Sistema Operativo

El primer paso es asegurar que el sistema operativo esté actualizado y preparado para la instalación de los servicios.

### 2.1 Actualización de Paquetes y Kernel

Ejecute los siguientes comandos para actualizar el sistema y aplicar parches de seguridad:

```bash
sudo apt update               # Refresca la lista de paquetes disponibles
sudo apt -y full-upgrade      # Aplica actualizaciones, incluyendo kernel y parches de seguridad
sudo reboot                   # Reinicia el servidor si se actualizó el kernel (recomendado)
```

### 2.2 Instalación de Utilidades Base

Instale las herramientas esenciales que se utilizarán durante el proceso de despliegue:

```bash
sudo apt -y install \
  unzip wget curl nano git cron \
  software-properties-common \
  apt-transport-https ca-certificates
```

> **ℹ️ Información:**
> *   `unzip`: Necesario para descomprimir el archivo de instalación de GLPI.
> *   `nano`: Editor de texto en consola para configuraciones rápidas.
> *   `software-properties-common`: Permite añadir repositorios PPA (necesario para PHP 8.0 en Ubuntu 22.04).

---

## 3. Instalación del Stack LAMP (Linux, Apache, MariaDB, PHP)

### 3.1 Selección y Configuración de PHP

La versión de PHP a instalar depende de la versión de Ubuntu:

| Ubuntu | PHP por Defecto | Acción Recomendada |
| :--- | :--- | :--- |
| 20.04 | 7.4 | Válido *out-of-the-box*. |
| 22.04 | 8.1 | Añadir PPA y **fijar la versión 8.0**. |

Si utiliza Ubuntu 22.04, ejecute los siguientes comandos para añadir el PPA de PHP:

```bash
# Comandos exclusivos para Ubuntu 22.04
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

### 3.2 Instalación de Apache y Extensiones de PHP 8.0

Instale el servidor web Apache y todas las extensiones de PHP requeridas por GLPI 9.5.4:

```bash
sudo apt -y install \
  apache2 \
  libapache2-mod-php8.0 \
  php8.0-{mysql,mbstring,xml,zip,curl,gd,intl,ldap,bz2,soap,imap,apcu,cli,common,opcache,xmlrpc}
```

> **⚙️ Extensiones Clave:**
> *   `mysql`: Conector esencial para MariaDB.
> *   `mbstring` / `xml`: Componentes internos críticos para el motor de GLPI.
> *   `ldap`: Requerido si se planea integrar con Active Directory (AD) u OpenLDAP.
> *   `apcu`: Módulo de caché en memoria que reduce las consultas a la base de datos, mejorando el rendimiento.

> Instalación Opcional de CAS
{.is-warning}


Instale el módulo CAS solo si planea utilizarlo para autenticación centralizada:

```bash
cd /tmp
wget https://github.com/apereo/phpCAS/archive/1.6.0.tar.gz
tar -xzf 1.6.0.tar.gz
sudo mkdir -p /usr/share/php/CAS
sudo cp -r phpCAS-1.6.0/source/* /usr/share/php/CAS/
```
Después registra la librería; crea `/etc/php/8.0/mods-available/cas.ini` correcto:
```bash
; CAS client for PHP (phpCAS)
include_path=/usr/share/php/CAS
```
```bash
sudo phpenmod cas
sudo systemctl restart apache2
```

#### Verificación de Extensiones de PHP

Verifique que todas las extensiones requeridas se hayan instalado correctamente:

```bash
php -m | grep -E '(mysql|mbstring|xml|zip|curl|gd|intl|ldap|soap|imap|apcu|opcache)'
```

> **✅ Verificación Exitosa:** El comando anterior debe listar todas las extensiones. Si falta alguna, instálela individualmente.
{.is-success}


### 3.3 Instalación de MariaDB

Instale el servidor de base de datos MariaDB:

```bash
sudo apt -y install mariadb-server mariadb-client
sudo systemctl enable --now mariadb   # Inicia el servicio y lo habilita para el arranque
```

---

## 4. Configuración de MariaDB

### 4.1 Script de Seguridad Inicial

Ejecute el script de seguridad para proteger la instalación de MariaDB:

```bash
sudo mysql_secure_installation
```

> **ℹ️ Respuestas Recomendadas:**
> *   **Contraseña `root`**: Establezca una contraseña robusta.
> *   **Cambiar a `unix_socket` authentication**: Sí.
> *   **Eliminar usuarios anónimos**: Sí.
> *   **Desactivar login remoto `root`**: Sí.
> *   **Eliminar BD de prueba**: Sí.
> *   **Recargar privilegios**: Sí.


### 4.2 Creación de Base de Datos y Usuario Específico

Acceda a la consola de MariaDB como usuario `root`:

```bash
sudo mysql -u root -p
```

Dentro de la consola SQL, ejecute los siguientes comandos para crear la base de datos y el usuario dedicado para GLPI:

```sql
-- Crear base de datos con codificación UTF8MB4
CREATE DATABASE glpi954
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
  
-- Crear usuario y asignar contraseña
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'Contraseña';

-- Asignar todos los privilegios sobre la BD 'glpi954' al nuevo usuario
GRANT ALL PRIVILEGES ON glpi954.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;

-- Salir de la consola SQL
EXIT;
```

> **🚨 Advertencia:** Reemplace `Contraseña` por una contraseña única y compleja. **Guarde esta contraseña** en un gestor de claves seguro.
{.is-warning}

---

## 5. Descarga y Despliegue de GLPI 9.5.4

### 5.1 Descarga Verificada

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/9.5.4/glpi-9.5.4.tgz
```

> **(Opcional)**: Comprueba hash

```bash
sha256sum glpi-9.5.4.tgz
```
### 5.2 Descompresión y permisos
```bash
sudo tar -xzf glpi-9.5.4.tgz -C /var/www/html
sudo mv /var/www/html/glpi /var/www/html/glpi954
sudo chown -R www-data:www-data /var/www/html/glpi954
sudo chmod -R 755 /var/www/html/glpi954
```

> 📂 Estructura resultante:
`/var/www/html/glpi954/` → index.php, inc/, front/, files/, etc.

## 6. Ajustes de PHP
Edita el php.ini que usa Apache:
```bash
sudo nano /etc/php/8.0/apache2/php.ini
```
> Parámetros críticos:
{.is-warning}
```bash
memory_limit        = 256M        # Evita agotar RAM al exportar PDFs grandes
upload_max_filesize = 16M
post_max_size       = 32M
max_execution_time  = 300
session.cookie_httponly = On     # Mitiga XSS
session.use_trans_sid   = 0
date.timezone           = "Europe/Madrid"  # ⬅ Ajusta a tu zona
opcache.enable          = 1
opcache.memory_consumption = 128
``` 
Guarda y reinicia Apache
```bash
sudo systemctl restart apache2
```
## 7. VirtualHost dedicado y Apache
### 7.1 Crear archivo de sitio
```bash
sudo nano /etc/apache2/sites-available/glpi954.conf
```
### 7.2 Activar sitio y módulos
```bash
sudo a2dissite 000-default      # Desactiva el default
sudo a2ensite  glpi954
sudo a2enmod   rewrite
sudo apache2ctl configtest      # Syntax OK ?
sudo systemctl reload apache2
```
> Ahora puede acceder a la URL para iniciar la instalación de la base de datos a través del asistente web:
>
> `http://<ip_servidor>:8095`
> 
| Paso | Acción                 | Comentario                                      |
| ---- | ---------------------- | ----------------------------------------------- |
| 1    | Idioma → Español       |                                                 |
| 2    | Licencia GPL           | Acepta                                          |
| 3    | Tipo instalación       | **Nueva instalación**                           |
| 4    | Requisitos             | Todo verde ✅                                    |
| 5    | Conexión BD            | `localhost`, `glpiuser`,Contraseña, `glpi954` |
| 6    | Creación esquema       | Automática                                      |
                       
---

 **¡Borra `/install`!** | Crítico por seguridad
```bash
sudo rm -rf /var/www/html/glpi954/install
```

## 9. Post-instalación (seguridad & cron)
### 9.1 Credenciales por defecto

| Usuario | Contraseña | Rol         |
| ------- | ---------- | ----------- |
| glpi    | glpi       | Super-Admin |
| tech    | tech       | Técnico     |
| normal  | normal     | Normal      |
| post-only  | post-only     | Sólo lectura      |

> 🔒 Cámbialas inmediatamente:
Administración > Usuarios > [usuario] > Contraseña
{.is-warning}
### 9.2 Tareas planificadas
GLPI necesita ejecutar cron.php cada 5 min para:
> *   Envío de mails
> *   Alertas de contratos
> *   Recolección de agentes
{.is-info}

```bash
sudo crontab -u www-data -e
```

Añade
```bash
# GLPI 9.5.4
*/5 * * * * /usr/bin/php /var/www/html/glpi954/front/cron.php &>/dev/null
```

### 9.3 Refuerzo de permisos
```bash
sudo chown -R www-data:www-data /var/www/html/glpi954
sudo find /var/www/html/glpi954 -type d -exec chmod 755 {} \;
sudo find /var/www/html/glpi954 -type f -exec chmod 644 {} \;
# Carpetas de escritura (files, config, marketplace)
sudo chmod -R 775 /var/www/html/glpi954/{files,config,marketplace}
```

## 10. Backups y mantenimiento
### 10.1 Script de backup diario

Crea `/opt/backup-glpi.sh`:
```bash
#!/bin/bash
DIR="/opt/backups"
DB="glpi954"
DBUSER="glpiuser"
DBPASS="Contraseña"

[ ! -d "$DIR" ] && mkdir -p "$DIR"

# Base de datos
mysqldump -u$DBUSER -p$DBPASS \
  --single-transaction --routines --events \
  $DB | gzip > $DIR/glpiDB-$(date +%F).sql.gz

# Archivos
tar -czf $DIR/glpiFiles-$(date +%F).tar.gz \
  -C /var/www/html glpi954 \
  --exclude=glpi954/files/_tmp \
  --exclude=glpi954/files/_cache

# Borrar backups > 7 días
find $DIR -type f -mtime +7 -delete
```

Hazlo ejecutable:
```bash
sudo chmod +x /opt/backup-glpi.sh
```
Tarea Planificada:
```bash
sudo crontab -e

# 02:30 cada día
30 02 * * * /opt/backup-glpi.sh
```

