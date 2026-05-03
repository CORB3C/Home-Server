# Home-Server
A small Server For Home Basic needs
# 🖥️ CRB Studio — Home Server Setup

> **MSI Mini PC · Ubuntu Server 24.04 LTS · Docker · Self-hosted services**

Documentación completa del proceso de montaje de un servidor casero multipropósito, desde cero hasta un sistema con almacenamiento en la nube, automatización del hogar, control de versiones para desarrollo de videojuegos y acceso remoto seguro.

Este es un proyecto que inicie mucho antes de ponerme manos a la obra con el tema de ciberseguridad. No obstante como en el futuro trabajare mucho con el me parecio interesante hacer un writeout sobre el proceso y documentar lo que fui aprendiendo.

Nota: la intencion era incluir los comandos de la operacion desde un principio, pero dado que contienen informacion personal como IPs y credenciales, creo que es mejor que se quede asi y solo documentare lo que hice, las conclusiones que saque y los problemas que encontre y tuve que resolver.

---

## 📋 Índice

1. [Hardware y sistema base](#1-hardware-y-sistema-base)
2. [Docker Engine](#2-docker-engine)
3. [Nextcloud — almacenamiento en la nube](#3-nextcloud--almacenamiento-en-la-nube)
4. [Nginx — reverse proxy](#4-nginx--reverse-proxy)
5. [SSL con Let's Encrypt / Certbot](#5-ssl-con-lets-encrypt--certbot)
6. [Home Assistant — automatización del hogar](#6-home-assistant--automatización-del-hogar)
7. [Perforce Helix Core — control de versiones para Unreal Engine](#7-perforce-helix-core--control-de-versiones-para-unreal-engine)
8. [Seguridad: Fail2ban, UFW y SSH](#8-seguridad-fail2ban-ufw-y-ssh)
9. [IP dinámica y DNS](#9-ip-dinámica-y-dns)
10. [Autologin tras corte de luz](#10-autologin-tras-corte-de-luz)
11. [Servidor de Minecraft con Cobblemon](#11-servidor-de-minecraft-con-cobblemon)
12. [ZeroTier VPN — acceso remoto](#12-zerotier-vpn--acceso-remoto)
13. [Arquitectura final](#13-arquitectura-final)
14. [Lecciones aprendidas](#14-lecciones-aprendidas)

---

## 1. Hardware y sistema base

**Hardware:**
- **Máquina:** MSI Mini PC
- **Arquitectura:** x86_64
- **OS:** Ubuntu Server 24.04 LTS con Interfaz Grafica para primeras configuraciones

- El servidor cuenta con una serie de dominios y subdominos, para acceder a los distintos servicios instalados. esto hace que solo tenga abiertos los puertos 80 y 443 (Http y Https)

**Filosofía de la infraestructura:**
- Todos los servicios corren en **Docker** para aislamiento y portabilidad
- **Nginx** instalado directamente en el host como reverse proxy
- **SSL** con Let's Encrypt para todos los subdominios
- **Fail2ban** + **UFW** para seguridad perimetral

---

## 2. Docker Engine

Se instaló Docker Engine mediante el repositorio oficial de apt (no Snap, para evitar rate limiting y problemas de compatibilidad con Nginx).

---

## 3. Nextcloud — almacenamiento en la nube

> **Nota importante:** Nextcloud Snap resulto ser incompatible con Nginx como reverse proxy — Snap bloqueaba conexiones HTTP externas con protecciones propias hardcodeadas. La solución que funciono fue Docker.

### docker-compose.yml

---

## 4. Nginx — reverse proxy

Nginx se instala directamente en el host (no en Docker) para gestionar el enrutado de todos los subdominios.


## 5. SSL con Let's Encrypt / Certbot


Los certificados se renuevan automáticamente vía timer de systemd. Validez: 90 días con auto-renovación.

---

## 6. Home Assistant — automatización del hogar

### docker-compose.yml

Home Assistant queda accesible en el puerto `8123` y se expone via Nginx en `home.crbstudio.com`.

### Configuración Nginx para Home Assistant (`/etc/nginx/sites-available/home.crbstudio.com`)


## 7. Perforce Helix Core — control de versiones para Unreal Engine

### ¿Por qué Perforce y no Git?

Git es inadecuado para proyectos Unreal Engine por el tamaño de los assets binarios (`.uasset`, `.umap`, etc.). Perforce gestiona archivos binarios grandes de forma nativa y es el estándar de la industria AAA.

**Stack elegido:** Perforce Helix Core (versión control completo) + itch.io (portfolio público)

### docker-compose.yml

## 8. Seguridad: Fail2ban, UFW y SSH

### SSH deshabilitado.

-Para evitar exponer el puerto SSH y atraer bots, inicialmente cambié el puerto por defecto. Sin embargo, encontré una solución más robusta: eliminar la regla del UFW y añadir un túnel VPN. De este modo, el servidor no queda expuesto a internet y puedo acceder por SSH con tranquilidad cuando estoy fuera de casa

```bash
sudo nano /etc/ssh/sshd_config
# Cambiar: Port 2222
sudo systemctl restart ssh
```

Deshabilitar login por contraseña:
```
PasswordAuthentication no
PubkeyAuthentication yes
```

### UFW — puertos abiertos

```bash
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 2222/tcp    # SSH
sudo ufw allow 1666/tcp    # Perforce
sudo ufw enable
```

### Fail2ban

```bash
sudo apt install fail2ban -y
```

`/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = 2222
maxretry = 5
bantime = 3600

[nextcloud]
enabled = true
port = 80,443
filter = nextcloud
logpath = /path/to/nextcloud/log/nextcloud.log
maxretry = 5
bantime = 3600
```

> **⚠️ Problema conocido:** las IPs dinámicas de redes móviles 5G cambian frecuentemente. Si se producen intentos fallidos de conexión SSH desde el móvil, Fail2ban puede banear la nueva IP. **Solución:** configurar autenticación por clave SSH en el cliente móvil (JuiceSSH en Android) para evitar intentos de contraseña fallidos.

---

## 9. IP dinámica y DNS

El ISP asigna una IP pública dinámica que puede cambiar en cualquier reinicio del router. Esto rompe los registros DNS A de los subdominios.

### Solución temporal (manual)

1. Acceder al router (p.ej. `192.168.1.1`)
2. Anotar la IP WAN actual
3. Actualizar los registros A en Cheapnames:

| Nombre | Tipo | Valor |
|---|---|---|
| `cloud` | A | nueva IP pública |
| `home` | A | nueva IP pública |

### Verificar propagación DNS

```bash
# Google DNS (más rápido que el resolver local)
curl "https://dns.google/resolve?name=cloud.crbstudio.com&type=A"
```

### Solución permanente (pendiente de implementar)

Cloudflare DDNS mediante contenedor Docker — actualiza automáticamente el DNS cuando cambia la IP:

```yaml
services:
  cloudflare-ddns:
    image: oznu/cloudflare-ddns:latest
    restart: unless-stopped
    environment:
      - API_KEY=TU_CLOUDFLARE_API_KEY
      - ZONE=crbstudio.com
      - SUBDOMAIN=cloud
```

> **Nota:** requiere migrar el DNS de Cheapnames a Cloudflare (gratuito) para disponer de API de actualización automática.

---

## 10. Autologin tras corte de luz

Sin autologin configurado, tras un corte de luz el servidor arranca pero se queda esperando login manual en TTY — los servicios Docker no levantan solos.

```bash
sudo systemctl edit getty@tty1
```

Pegar en el editor:
```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin manuadmin --noclear %I $TERM
```

```bash
sudo systemctl daemon-reload
```

Tras esto, el usuario `manuadmin` hace login automáticamente al arrancar, y los contenedores Docker con `restart: unless-stopped` levantan solos.

---

## 11. Servidor de Minecraft con Cobblemon

Servidor Fabric + Cobblemon para jugar en red local/remota.

### docker-compose.yml

```yaml
services:
  minecraft:
    image: itzg/minecraft-server:java21
    container_name: minecraft
    restart: unless-stopped
    ports:
      - "25565:25565"
    environment:
      EULA: "TRUE"
      TYPE: FABRIC
      VERSION: "1.21.1"
      MEMORY: "4G"
      MODRINTH_PROJECTS: "cobblemon,fabric-api"
    volumes:
      - ./data:/data
```

```bash
cd ~/docker-services/minecraft
docker compose up -d
docker logs -f minecraft
```

El servidor está listo cuando aparece: `Done! For help, type "help"`

> **Notas:**
> - Usar la imagen `java21` — Cobblemon no es compatible con Java 25
> - `MODRINTH_PROJECTS` descarga los mods automáticamente desde Modrinth
> - Para 3 jugadores, 4GB de RAM es suficiente

---

## 12. ZeroTier VPN — acceso remoto

ZeroTier crea una red virtual privada que permite acceder al servidor como si estuvieras en la red local, sin exponer puertos SSH directamente a internet.

### Por qué ZeroTier y no WireGuard

WireGuard requiere configuración manual más compleja. ZeroTier ofrece:
- Setup más sencillo
- Clientes para Linux, Windows y Android
- Panel web para gestión de dispositivos
- Red mesh (todos los dispositivos se ven entre sí)

### Instalación en el servidor (Linux)

```bash
curl -s https://install.zerotier.com | sudo bash
sudo zerotier-cli join TU_NETWORK_ID
```

### Instalación en Windows

Descargar desde [zerotier.com/download](https://www.zerotier.com/download/)

### Instalación en Android

App ZeroTier One disponible en Google Play.

### Autorizar dispositivos

En el panel web [my.zerotier.com](https://my.zerotier.com):
1. Crear una nueva red
2. Anotar el Network ID de 16 caracteres
3. Cada dispositivo que se une aparece como "pending" — marcarlos como autorizados

### Cerrar SSH al exterior tras configurar ZeroTier

```bash
# Eliminar regla que permite SSH desde cualquier IP
sudo ufw delete allow 2222/tcp

# Permitir SSH solo desde la red ZeroTier (rango 10.x.x.x por defecto)
sudo ufw allow from 10.0.0.0/8 to any port 2222
```

> **⚠️ Verificar siempre el acceso SSH por ZeroTier ANTES de cerrar el puerto exterior.**

---

## 13. Arquitectura final

```
Internet
    │
    ├─── 80/443 (HTTP/HTTPS) ──► Nginx (host)
    │                                 ├─► Nextcloud (Docker :9000)
    │                                 └─► Home Assistant (Docker :8123)
    │
    └─── ZeroTier VPN ──────────────► SSH :2222
                                  ├─► Perforce :1666
                                  └─► Todos los servicios internos

Seguridad:
    UFW ──► Fail2ban ──► SSH (clave pública, no contraseña)
                    └─► Nextcloud (logs de acceso)
```

### Servicios en producción

| Servicio | Imagen Docker | Puerto interno | Acceso externo |
|---|---|---|---|
| Nextcloud | `nextcloud:latest` | 9000 | `cloud.crbstudio.com` |
| Home Assistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | `home.crbstudio.com` |
| Perforce Helix Core | `sourcegraph/helix-p4d:2023.1` | 1666 | Via ZeroTier |
| Minecraft + Cobblemon | `itzg/minecraft-server:java21` | 25565 | Via ZeroTier |

---

## 14. Lecciones aprendidas

**Docker vs Snap**
Nextcloud Snap bloquea conexiones HTTP externas con protecciones hardcodeadas que son incompatibles con Nginx como reverse proxy. Docker es siempre la opción correcta para servicios que necesitan un reverse proxy.

**IP dinámica**
Las IPs dinámicas rompen silenciosamente los servicios cuando el router reinicia. DDNS automatizado (Cloudflare) es imprescindible para evitar actualizaciones manuales.

**Fail2ban + móvil**
Las redes 5G cambian de IP con frecuencia. Si Fail2ban detecta intentos fallidos de SSH desde una IP de móvil, banea esa IP. La autenticación por clave SSH en el cliente móvil elimina este problema porque nunca hay intentos fallidos de contraseña.

**Git no es para Unreal Engine**
Los `.uasset` y `.umap` son archivos binarios que pueden superar los 100MB. Git no los gestiona bien (LFS es parcheado y tiene limitaciones). Perforce es el estándar de la industria para proyectos con assets grandes.

**Autologin en servidor headless**
Un servidor sin pantalla necesita autologin configurado en TTY para que los servicios arranquen automáticamente tras un corte de luz. Sin esto, el servidor queda parado esperando login manual.

**microk8s / Kubernetes**
Si tienes microk8s instalado, sus reglas de iptables-legacy pueden interferir con HTTPS desde dentro de contenedores Docker. Si no usas Kubernetes activamente, desinstalarlo simplifica la configuración de red.

**.p4ignore en Windows**
Windows puede guardar el archivo como `.p4ignore.txt` aunque parezca que se llama `.p4ignore`. Verificar con `dir /a` en CMD y renombrar si es necesario.

---

## 📁 Estructura de archivos de configuración

```
~/docker-services/
├── nextcloud/
│   └── docker-compose.yml
├── homeassistant/
│   ├── docker-compose.yml
│   └── config/
├── perforce/
│   ├── docker-compose.yml
│   └── data/
└── minecraft/
    ├── docker-compose.yml
    └── data/

/etc/nginx/sites-available/
├── cloud.crbstudio.com
└── home.crbstudio.com

/etc/fail2ban/
└── jail.local
```

---

*CRB Studio — servidor casero en producción continua desde 2024*
