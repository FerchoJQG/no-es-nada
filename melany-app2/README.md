# ⚙️ Melany — APP 2

**IP de tu VM:** `192.168.107.3`  
**Rol:** Servidor de Aplicación 2 (Node.js en puerto 3000)

---

## 📋 Resumen de lo que vas a hacer

1. Configurar IP estática con VLAN 107
2. Instalar Node.js con NVM
3. Instalar PM2
4. Clonar la aplicación desde GitHub
5. Configurar variables de entorno (conexión a la DB de Limbert)
6. Lanzar la app con PM2
7. Instalar Node Exporter (para monitoreo)

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
        - "192.168.100.177/24"
  vlans:
    vlan107:
      id: 107
      link: ens18
      addresses:
        - "192.168.107.3/29"
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: default
          via: 192.168.107.1
```

```bash
sudo netplan apply
```

Verifica tu IP:

```bash
ip addr show
```

---

## Paso 2 — Cambiar el Hostname

```bash
sudo hostnamectl set-hostname app2
```

Cierra y vuelve a abrir la terminal para ver el cambio.

---

## Paso 3 — Instalar Node.js con NVM

Descarga e instala NVM:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

Carga NVM en la sesión actual (sin cerrar la terminal):

```bash
\. "$HOME/.nvm/nvm.sh"
```

Instala Node.js v22:

```bash
nvm install 22
```

Verifica las versiones:

```bash
node -v
npm -v
```

---

## Paso 4 — Instalar PM2

```bash
npm install pm2@latest -g
```

Verifica la instalación:

```bash
pm2 --version
```

---

## Paso 5 — Clonar la Aplicación

```bash
mkdir ~/apps && cd ~/apps
```

```bash
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app2_3000
```

Instala las dependencias:

```bash
cd ~/apps/app2_3000 && npm install
```

---

## Paso 6 — Configurar Variables de Entorno

```bash
cd ~/apps/app2_3000 && cp .env.example .env
```

```bash
nano ~/apps/app2_3000/.env
```

Asegúrate de que el archivo tenga exactamente esto:

```env
PORT=3000
DB_HOST=192.168.107.2
DB_USER=usr_movies
DB_PASSWORD=secret
DB_NAME=db_movies
```

Guarda con `Ctrl+O`, `Enter`, `Ctrl+X`.

---

## Paso 7 — Prueba Manual (antes de PM2)

> ⚠️ Asegúrate de que Limbert ya terminó de configurar la base de datos antes de este paso.

```bash
cd ~/apps/app2_3000 && node app.js
```

Deberías ver:

```
Servidor ejecutándose en el puerto 3000
Conexión a MariaDB exitosa. Pool creado y probado.
```

Si aparece ese mensaje, todo está bien. Detén la app con `Ctrl+C`.

Si hay error de conexión, verifica con Limbert que la DB esté lista.

---

## Paso 8 — Lanzar con PM2

```bash
cd ~/apps/app2_3000 && pm2 start app.js --name app2_3000
```

Verifica que está corriendo:

```bash
pm2 status
```

Configura auto-arranque al reiniciar:

```bash
pm2 startup
```

Ejecuta el comando que PM2 te indica (cópialo y pégalo).

Guarda la configuración:

```bash
pm2 save
```

---

## Paso 9 — Instalar Node Exporter (para monitoreo)

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
# La app está corriendo en PM2
pm2 status

# La app responde en el puerto 3000
curl http://localhost:3000/movies

# La app responde desde la IP de la VLAN
curl http://192.168.107.3:3000/movies

# Node Exporter está corriendo
sudo systemctl status prometheus-node-exporter
```

---

## 🔧 Comandos Útiles de PM2

```bash
# Ver logs de la app
pm2 logs app2_3000

# Reiniciar la app
pm2 restart app2_3000

# Detener la app (para prueba de failover)
pm2 stop app2_3000

# Volver a iniciar
pm2 start app2_3000
```

---

## 🎯 Durante la Prueba de Failover

Cuando el profesor pida demostrar que el balanceador sigue funcionando si una app cae:

```bash
# Detén tu app (el proxy de Fernando debería redirigir todo a Daner)
pm2 stop app2_3000

# Cuando termines la demo, vuelve a levantarla
pm2 start app2_3000
```
