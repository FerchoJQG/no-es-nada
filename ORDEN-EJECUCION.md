# 📋 Orden de Ejecución del Grupo

Este documento define el orden correcto para configurar todo sin bloqueos.

---

## 🕐 Fase 1 — Todos al mismo tiempo

Cada uno configura su IP estática con VLAN 107. Esto lo hacen todos en paralelo.

| Quién | Comando clave | IP resultante |
|-------|---------------|---------------|
| Fernando | `sudo netplan apply` | `192.168.107.2` |
| Limbert | `sudo netplan apply` | `192.168.107.2` |
| Melany | `sudo netplan apply` | `192.168.107.3` |
| Daner | `sudo netplan apply` | `192.168.107.4` |

Verificación de conectividad (todos hacen ping entre sí):

```bash
# Fernando verifica que llega a todos
ping -c 3 192.168.107.2   # Limbert
ping -c 3 192.168.107.3   # Melany
ping -c 3 192.168.107.4   # Daner
```

---

## 🕑 Fase 2 — Limbert primero (DB)

**Limbert** levanta la base de datos primero porque las apps dependen de ella.

1. Instalar MariaDB
2. Hardening
3. Configurar bind-address
4. Crear `db_movies`, usuario `usr_movies`, tabla `movies` e insertar datos

**Señal para el grupo:** Cuando Limbert diga "DB lista", Melany y Daner pueden continuar.

---

## 🕒 Fase 3 — Melany y Daner en paralelo (Apps)

Con la DB lista, **Melany** y **Daner** instalan Node.js, PM2, clonan el repo y configuran el `.env`.

Antes de lanzar con PM2, cada una hace la prueba manual:
```bash
node app.js
# Debe aparecer: "Conexión a MariaDB exitosa"
```

Luego lanzar con PM2:
```bash
pm2 start app.js --name app1_3000   # Daner
pm2 start app.js --name app2_3000   # Melany
```

**Señal para el grupo:** Cuando ambas digan "App corriendo", Fernando puede verificar el balanceo.

---

## 🕓 Fase 4 — Fernando verifica el balanceo

```bash
curl http://192.168.107.4:3000/movies   # App 1 directo
curl http://192.168.107.3:3000/movies   # App 2 directo
curl http://192.168.107.2/movies        # A través del proxy (balanceo)
```

---

## 🕔 Fase 5 — Fernando instala monitoreo

Instala Prometheus + Grafana, configura los scrape targets y verifica que las 4 VMs aparecen en Grafana.

---

## 🕕 Fase 6 — Verificación Final del Grupo

Prueba de failover:
1. Melany detiene su app: `pm2 stop app2_3000`
2. Fernando hace varias peticiones al proxy: el 100% debe ir a Daner
3. Melany levanta su app: `pm2 start app2_3000`
4. Las peticiones vuelven a distribuirse entre ambas

---

## ⚠️ Dependencias Críticas

```
Limbert (DB)
    └──→ Melany (App 2) depende de DB
    └──→ Daner (App 1) depende de DB
              └──→ Fernando (Proxy) depende de Apps
                        └──→ Fernando (Monitoreo) depende de todo
```

**Si algo falla, verificar en este orden:**
1. ¿Hay conectividad de red? (ping entre VMs)
2. ¿La DB de Limbert está escuchando en `192.168.107.2:3306`?
3. ¿Las apps de Melany y Daner responden en `:3000`?
4. ¿El proxy de Fernando balancea correctamente?
