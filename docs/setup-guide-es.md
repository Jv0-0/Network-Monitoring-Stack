# 🛠️ Guía de Instalación — Network Monitoring Stack

> Guía práctica para levantar el stack completo (Zabbix, Grafana, n8n, Open WebUI) sobre WSL2 + Docker, conectado a la topología de PNetLab vía Tailscale.

---

## ✅ Requisitos previos

Antes de empezar, asegúrate de tener:

- **Windows 10/11** con **WSL2** habilitado
- **Ubuntu 22.04+** instalado sobre WSL2
- **Docker** + **Docker Compose v2** funcionando dentro de WSL2
- Una cuenta gratuita de **[Tailscale](https://tailscale.com)**
- **PNetLab CE** corriendo (en VM local o en la nube)
- Un bot de **Telegram** creado (vía [@BotFather](https://t.me/BotFather)) si vas a usar las alertas

---

## 1️⃣ Clonar el repositorio

```bash
git clone https://github.com/<tu-usuario>/network-monitoring-stack.git
cd network-monitoring-stack
```

---

## 2️⃣ Configurar las variables de entorno

Dentro de `Docker-Stack/docker composes/` copia el archivo de ejemplo:

```bash
cd "Docker-Stack/docker composes"
cp env.example .env
nano .env
```

Completa cada valor:

```env
# PostgreSQL
POSTGRES_DB=zabbix
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=tu_password_segura

# Grafana
GF_ADMIN_USER=admin
GF_ADMIN_PASSWORD=tu_password_segura

# n8n
N8N_USER=admin
N8N_PASSWORD=tu_password_segura
N8N_WEBHOOK_URL=https://tu-subdominio.ngrok-free.dev

# Tailscale (genera tu key en tailscale.com/admin/settings/keys)
TS_AUTHKEY=tskey-auth-tu-key-aqui

# Open WebUI / OpenRouter (genera tu key en openrouter.ai/keys)
OPENROUTER_API_KEY=sk-or-v1-tu-key-aqui
OPENWEBUI_SECRET_KEY=cualquier-string-aleatorio
```

> ⚠️ **Nunca subas tu `.env` real a GitHub.** Solo el `.env.example` debe quedar en el repositorio.

---

## 3️⃣ Levantar el stack con Docker Compose

```bash
docker compose up -d
```

Verifica que todos los contenedores estén corriendo:

```bash
docker compose ps
```

Deberías ver algo similar a:

```
NAME             STATUS
tailscale        Up
nginx            Up
postgres         Up
zabbix-server    Up
zabbix-web       Up
grafana          Up
n8n              Up
open-webui       Up
```

---

## 4️⃣ Acceder a los servicios

| Servicio | URL local | Credenciales |
|---|---|---|
| Zabbix | `http://localhost:8080` | Admin / zabbix (cambiar al primer login) |
| Grafana | `http://localhost:3000` | admin / (la que pusiste en `.env`) |
| n8n | `http://localhost:5678` | admin / (la que pusiste en `.env`) |
| Open WebUI | `http://localhost:3001` | crear cuenta al primer ingreso |

---

## 5️⃣ Conectar Tailscale al lab de PNetLab

En la VM de PNetLab, instala Tailscale y anuncia las subredes internas del lab:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=192.168.10.0/24,192.168.20.0/24,192.168.30.0/24 --accept-routes
```

Luego, en el [panel de administración de Tailscale](https://login.tailscale.com/admin/machines), aprueba las rutas anunciadas (subnet routes).

---

## 6️⃣ Importar la topología en PNetLab

1. Copia el archivo `.unl` desde `pnetlab/Topology/` a tu servidor PNetLab
2. Impórtalo desde la interfaz web de PNetLab
3. En cada router Cisco, habilita SNMP v2c con la comunidad `public`:

```cisco
snmp-server community public RO
snmp-server location PNetLab-Lab
```

4. Si el router hace NAT (como Manga), revisa que la ACL de NAT no esté bloqueando el tráfico de Tailscale ni de Docker (ver detalles en el `README.md`, sección de ACLs).

---

## 7️⃣ Agregar los hosts en Zabbix

1. Ve a **Configuration → Hosts → Create Host**
2. Asigna la IP de Tailscale de cada router como interfaz SNMP
3. Vincula el template `Template Net Cisco IOS SNMPv2`
4. Repite para cada dispositivo de la topología

> 💡 Puedes importar directamente el archivo `Zabbix/zbx_export_hosts.xml` desde **Configuration → Hosts → Import** para no configurar cada host manualmente.

---

## 8️⃣ Conectar Grafana con Zabbix

1. Instala el plugin si no quedó incluido automáticamente:

```bash
docker exec -it grafana grafana-cli plugins install alexanderzobnin-zabbix-datasource
docker compose restart grafana
```

2. En Grafana, ve a **Connections → Data sources → Add data source**
3. Selecciona **Zabbix**
4. URL: `http://zabbix-web:8080/api_jsonrpc.php`
5. Usuario y contraseña: los de tu cuenta admin de Zabbix
6. Importa el dashboard desde `Graffana/dashboards/network_overview.json`

---

## 9️⃣ Configurar las alertas de Telegram con n8n

1. En n8n, importa el workflow desde `N8N/workflows/Automatization-n8n.json`
2. Configura tus credenciales del bot de Telegram dentro del nodo correspondiente
3. En Zabbix, crea un **Media Type** tipo Webhook apuntando a:
   ```
   http://n8n:5678/webhook/zabbix-alert
   ```
4. Asigna ese media type a tu usuario y crea las acciones de alerta según severidad

> 📌 Durante el desarrollo se usó **ngrok** para exponer el webhook de n8n públicamente mientras se hacían pruebas. Para uso más estable, Tailscale puede manejar esta conectividad sin depender de ngrok.

---

## 🔟 Verificar que todo funciona

- [ ] Los 21 hosts aparecen como disponibles ("Up") en Zabbix
- [ ] El dashboard de Grafana muestra datos en tiempo real
- [ ] Una alerta de prueba en Zabbix llega correctamente a tu Telegram
- [ ] Open WebUI responde preguntas sobre el estado de la red usando el JSON de Zabbix

---

## 🐛 Problemas comunes

**SNMP no responde aunque el ping funciona:**
Revisa la ACL de NAT en los routers que hacen NAT (Manga). Probablemente está enmascarando el tráfico antes de llegar al proceso SNMP. Ver sección de ACLs en el `README.md` para el detalle exacto.

**Tailscale pierde conexión tras reiniciar WSL2:**
```bash
docker compose down
wsl --shutdown
# Vuelve a abrir WSL2
docker compose up -d
```

**El plugin de Zabbix no carga en Grafana:**
Agrega esta variable en tu `.env` y reinicia el contenedor:
```env
GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS=alexanderzobnin-zabbix-datasource
```

**n8n no recibe las alertas del webhook de Zabbix:**
Verifica que ambos contenedores estén en la misma red de Docker:
```bash
docker exec zabbix-server curl http://n8n:5678/healthz
```

---

*Para más detalle técnico sobre la arquitectura, las ACLs, las VLANs y los desafíos resueltos, consulta el `README.md` principal del repositorio.*
