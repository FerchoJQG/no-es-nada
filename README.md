# 🚀 Lab 4.1 — Plataforma HA, Balanceo de Carga y Monitoreo

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes — SIS313  
**Docente:** Ing. Marcelo Quispe Ortega | **Semestre:** 1/2026

---

## 👥 Integrantes y Roles

| Alumno | IP | Hostname | Rol |
|--------|----|----------|-----|
| **Fernando** | `192.168.100.175` | `server-175` | 🔀 PROXY + MONITOREO |
| **Limbert** | `192.168.100.176` | `server-176` | 🗄️ BASE DE DATOS |
| **Melany** | `192.168.100.177` | `server-177` | ⚙️ APP 2 |
| **Daner** | `192.168.100.178` | `server-178` | ⚙️ APP 1 |

**VLAN ID:** `107`  
**Subred asignada:** `192.168.107.0/29`  
**Gateway:** `192.168.107.1`

---

## 🗺️ Arquitectura

```
Internet
    │
    ▼
┌─────────────────────────┐
│  PROXY + MONITOREO      │  192.168.107.2  (Fernando)
│  Nginx + Prometheus     │
│  + Grafana              │
└────────┬────────────────┘
         │ VLAN 107
    ┌────┴────┐
    ▼         ▼
┌────────┐  ┌────────┐
│ APP 1  │  │ APP 2  │
│ :3000  │  │ :3000  │
│ Daner  │  │ Melany │
│.107.4  │  │.107.3  │
└───┬────┘  └───┬────┘
    └─────┬─────┘
          ▼
    ┌──────────┐
    │    DB    │
    │ MariaDB  │
    │ Limbert  │
    │ .107.2   │
    └──────────┘
```

> ⚠️ **Nota:** En la práctica grupal, cada alumno trabaja en su propia VM física en el centro de datos.

---

## 📁 Guías por Rol

Cada carpeta contiene la guía paso a paso para cada integrante:

| Carpeta | Integrante | Contenido |
|---------|-----------|-----------|
| [`/fernando-proxy`](./fernando-proxy/README.md) | Fernando | Nginx, Prometheus, Grafana |
| [`/limbert-db`](./limbert-db/README.md) | Limbert | MariaDB, usuarios, tablas |
| [`/melany-app2`](./melany-app2/README.md) | Melany | Node.js App 2, PM2 |
| [`/daner-app1`](./daner-app1/README.md) | Daner | Node.js App 1, PM2 |

---

## 🌐 URLs de Acceso

| Servicio | URL |
|----------|-----|
| **Aplicación (Proxy)** | https://vlan107-app.rootcode.com.bo |
| **Grafana (Monitoreo)** | https://vlan107-monitoring.rootcode.com.bo |

---

## ✅ Checklist General del Grupo

- [ ] Todas las VMs tienen IP estática con VLAN 107 configurada
- [ ] Conectividad entre todas las VMs (ping entre sí)
- [ ] MariaDB acepta conexiones desde App 1 y App 2
- [ ] App 1 responde en `192.168.107.4:3000`
- [ ] App 2 responde en `192.168.107.3:3000`
- [ ] Nginx balancea entre App 1 y App 2
- [ ] Prometheus scrapea métricas de las 4 VMs
- [ ] Grafana muestra dashboards con datos reales
- [ ] Failover funciona al detener una app

---

## 🔑 Credenciales del Laboratorio

| Servicio | Usuario | Contraseña |
|----------|---------|------------|
| Grafana (primer ingreso) | `admin` | `admin` |
| MariaDB app user | `usr_movies` | `secret` |
| MariaDB root | `root` | *(la que configures en hardening)* |
