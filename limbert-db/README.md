# 🗄️ Limbert — BASE DE DATOS

**IP de tu VM:** `192.168.107.5`  
**Rol:** Servidor de Base de Datos (MariaDB)

> ⚠️ **Nota:** En la práctica grupal tu IP en la VLAN es `192.168.107.5`. Consulta la tabla de asignación de tu docente para confirmar.

---

## 📋 Resumen de lo que vas a hacer

1. Configurar IP estática con VLAN 107
2. Instalar MariaDB
3. Hacer hardening de seguridad
4. Configurar acceso remoto desde la VLAN
5. Crear la base de datos, usuario y tablas
6. Instalar Node Exporter (para monitoreo)

---

## Paso 1 — Configurar IP Estática con VLAN 107

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Reemplaza todo el contenido con esto:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      optional: true
      addresses:
        - "192.168.100.176/24"
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 8.8.8.8
  vlans:
    vlan107:
      id: 107
      link: ens18
      addresses:
        - "192.168.107.5/29"

```

```bash
sudo netplan apply
```

Verifica tu IP:

```bash
ip addr show
```

---

## Paso 2 — Instalar MariaDB

```bash
sudo apt update && sudo apt install mariadb-server -y
```

---

## Paso 3 — Hardening de MariaDB

```bash
sudo mysql_secure_installation
```

Responde así a cada pregunta:

```
Enter current password for root (enter for none):     [ENTER]
Switch to unix_socket authentication [Y/n]:            [ENTER]
Change the root password? [Y/n]:                       [ENTER]
New password:                                          [escribe tu contraseña]
Re-enter new password:                                 [repite la contraseña]
Remove anonymous users? [Y/n]:                         [ENTER]
Disallow root login remotely? [Y/n]:                   [ENTER]
Remove test database and access to it? [Y/n]:          [ENTER]
Reload privilege tables now? [Y/n]:                    [ENTER]
```

Prueba que puedes conectarte con root:

```bash
mysql -u root -h localhost -p
```

```sql
quit
```

---

## Paso 4 — Configurar Acceso Remoto

Edita la configuración de MariaDB para escuchar en la IP de la VLAN:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Busca la línea `bind-address = 127.0.0.1` y cámbiala por:

```
bind-address = 192.168.107.2
```

Reinicia MariaDB:

```bash
sudo systemctl restart mariadb
sudo systemctl enable mariadb
```

---

## Paso 5 — Crear Base de Datos, Usuario y Permisos

Accede al CLI de MariaDB:

```bash
mysql -u root -h localhost -p
```

Ejecuta estos comandos **uno por uno**:

```sql
CREATE DATABASE db_movies;
```

```sql
CREATE USER 'usr_movies'@'192.168.107.4' IDENTIFIED BY 'secret';
```

```sql
CREATE USER 'usr_movies'@'192.168.107.3' IDENTIFIED BY 'secret';
```

```sql
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'192.168.107.4';
```

```sql
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'192.168.107.3';
```

```sql
FLUSH PRIVILEGES;
```

```sql
quit
```

---

## Paso 6 — Crear Tabla e Insertar Datos

Conéctate con el nuevo usuario:

```bash
mysql -u usr_movies -h 192.168.107.2 -p
```

Contraseña: `secret`

Selecciona la base de datos y crea la tabla:

```sql
USE db_movies;
```

```sql
CREATE TABLE movies (
    id serial PRIMARY KEY,
    title character varying(150) NOT NULL,
    year integer,
    UNIQUE(title)
);
```

Inserta los datos de ejemplo:

```sql
INSERT INTO movies (title, year) VALUES
    ('Inception', 2010),
    ('The Matrix', 1999),
    ('Pulp Fiction', 1994),
    ('The Dark Knight', 2008),
    ('Eternal Sunshine of the Spotless Mind', 2004),
    ('Forrest Gump', 1994),
    ('Fight Club', 1999),
    ('The Godfather', 1972),
    ('Interstellar', 2014),
    ('Parasite', 2019);
```

Verifica que los datos se insertaron:

```sql
SELECT * FROM movies;
```

```sql
quit
```

---

## Paso 7 — Hardening UFW (Firewall)

Instala UFW si no está instalado:

```bash
sudo apt install ufw -y
```

Configura las reglas para permitir MariaDB solo desde las Apps:

```bash
# Permitir SSH (para no perder acceso)
sudo ufw allow ssh

# Permitir MariaDB solo desde App 1 (Daner) y App 2 (Melany)
sudo ufw allow from 192.168.107.4 to any port 3306
sudo ufw allow from 192.168.107.3 to any port 3306

# Permitir Node Exporter desde el Proxy (Fernando - monitoreo)
sudo ufw allow from 192.168.107.2 to any port 9100

# Activar UFW
sudo ufw enable
```

Verifica las reglas:

```bash
sudo ufw status verbose
```

---

## Paso 8 — Instalar Node Exporter (para monitoreo con Prometheus)

```bash
sudo apt install prometheus-node-exporter -y
```

Verifica que expone métricas:

```bash
curl http://localhost:9100/metrics | head -20
```

---

## ✅ Verificaciones Finales

```bash
# MariaDB está corriendo
sudo systemctl status mariadb

# Puedes conectarte desde la misma VM
mysql -u usr_movies -h 192.168.107.2 -p -e "USE db_movies; SELECT COUNT(*) FROM movies;"

# Node Exporter está corriendo
sudo systemctl status prometheus-node-exporter

# Verifica el puerto 3306 está escuchando en la IP correcta
ss -tlnp | grep 3306
```

---

## 🔧 Solución de Problemas Comunes

**Error: Can't connect to MySQL server on '192.168.107.2'**
```bash
# Verifica que bind-address esté bien configurado
grep bind-address /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb
```

**Error de permisos desde App 1 o App 2**
```bash
# Verifica los usuarios creados
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user='usr_movies';"
```
