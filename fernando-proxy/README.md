# 🔀 Fernando — PROXY + MONITOREO

**IP de tu VM:** `192.168.107.2`  
**Rol:** Proxy Inverso (Nginx) + Monitoreo (Prometheus + Grafana)

---

## 📋 Resumen de lo que vas a hacer

1. Configurar IP estática con VLAN 107
2. Instalar y configurar Nginx como balanceador de carga
3. Configurar enrutamiento NAT para las demás VMs
4. Instalar Prometheus y Node Exporter
5. Instalar Grafana y conectarlo a Prometheus

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
        - "192.168.100.175/24"
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
        - "192.168.107.2/29"
```

```bash
sudo netplan apply
```

Verifica que tienes las dos IPs:

```bash
ip addr show
```

---

## Paso 2 — Instalar Nginx

```bash
sudo apt update && sudo apt install nginx -y
```

---

## Paso 3 — Configurar el Balanceador de Carga

```bash
sudo nano /etc/nginx/sites-available/default
```

Reemplaza todo el contenido con esto:

```nginx
upstream loadbalancer {
    server 192.168.107.3:3000;  # App 1 - Daner
    server 192.168.107.4:3000;  # App 2 - Melany
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://loadbalancer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Verifica que la config es válida:

```bash
sudo nginx -t
```

Reinicia Nginx:

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

---

## Paso 4 — Enrutamiento NAT (para que Apps y DB tengan internet)

Habilitar reenvío de paquetes:

```bash
sudo nano /etc/sysctl.conf
```

Asegúrate de que esta línea esté **sin comentar**:

```
net.ipv4.ip_forward=1
```

Aplica el cambio:

```bash
sudo sysctl -p
```

Configura la regla NAT (reemplaza `ens18` si tu interfaz con internet tiene otro nombre):

```bash
sudo iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
```

Instala y guarda las reglas para que persistan:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

## Paso 5 — Instalar Prometheus y Node Exporter

```bash
sudo apt install prometheus prometheus-node-exporter -y
```

Verifica que las métricas están disponibles:

```bash
curl http://localhost:9100/metrics | head -20
```

---

## Paso 6 — Configurar Prometheus (scrapear las 4 VMs)

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Reemplaza la sección `scrape_configs` con esto:

```yaml
scrape_configs:
  - job_name: 'node-proxy'
    static_configs:
      - targets: ['192.168.107.2:9100']

  - job_name: 'node-app1'
    static_configs:
      - targets: ['192.168.107.3:9100']

  - job_name: 'node-app2'
    static_configs:
      - targets: ['192.168.107.4:9100']

  - job_name: 'node-db'
    static_configs:
      - targets: ['192.168.107.5:9100']
```

Reinicia Prometheus:

```bash
sudo systemctl restart prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

---

## Paso 7 — Instalar Grafana

Instala dependencias:

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
```

Importa la clave GPG:

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

Agrega el repositorio:

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

Instala Grafana:

```bash
sudo apt update
sudo apt install grafana -y
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Verifica que está corriendo:

```bash
sudo systemctl status grafana-server
```

---

## Paso 8 — Configurar Grafana (desde el navegador)

1. Accede a: **https://vlan107-monitoring.rootcode.com.bo**
2. Usuario: `admin` | Contraseña: `admin` → cambia la contraseña cuando te lo pida
3. Ve a **Connections → Data Sources → Add data source → Prometheus**
4. URL: `http://localhost:9090` → **Save & Test**
5. Ve a **Dashboards → Import**
   - Importa el ID `1860` (Node Exporter Full)
   - Importa el ID `10242` (Node Exporter Full with Node Name)

---

## ✅ Verificaciones Finales

```bash
# Nginx está corriendo
sudo systemctl status nginx

# Prometheus está corriendo
sudo systemctl status prometheus

# Grafana está corriendo
sudo systemctl status grafana-server

# Puedes llegar a App 1 y App 2 directamente
curl http://192.168.107.4:3000/movies
curl http://192.168.107.3:3000/movies

# El balanceador distribuye entre ambas
curl http://192.168.107.2/movies
curl http://192.168.107.2/movies
curl http://192.168.107.2/movies
```

---

## 🎯 Prueba de Failover (Demostración Final)

Cuando Daner o Melany detengan su app con `pm2 stop`, el proxy debe seguir respondiendo desde la otra instancia:

```bash
# Realiza varias peticiones seguidas y observa el campo "app" en la respuesta
for i in $(seq 1 10); do curl -s http://192.168.107.2/movies | python3 -c "import sys,json; d=json.load(sys.stdin); print(d)" 2>/dev/null || curl http://192.168.107.2/movies; done
```
