# n8n en Docker — Guía de instalación

Guía paso a paso para autoalojar n8n usando Docker y Docker Compose en un VPS con Linux.

---

## Requisitos previos

- VPS con Ubuntu 22.04 o superior
- Acceso SSH con usuario `sudo`
- Un dominio apuntando a la IP del servidor (para SSL)
- Docker 20.10+ instalado (ver Paso 1)

---

## Paso 1 — Instalar Docker

Desde Docker v20.10, Compose viene incluido. No es necesario instalarlo por separado.

```bash
# Verificar si Docker ya está instalado
docker -v
```

Si no está instalado, seguir la guía oficial:
https://docs.docker.com/engine/install/ubuntu/

> **Tip:** Algunos proveedores VPS (como Hostinger Docker VPS) incluyen Docker preinstalado. En ese caso, saltar este paso.

---

## Paso 2 — Preparar el directorio de datos

```bash
# Crear el directorio principal y navegar a él
mkdir ~/n8n && cd ~/n8n

# Crear el subdirectorio para datos persistentes
mkdir n8n_data

# Asignar permisos correctos (requerido por n8n en Docker)
sudo chown -R 1000:1000 n8n_data
```

La carpeta `n8n_data` mapeará los datos de n8n fuera del contenedor, garantizando que sobrevivan a reinicios y actualizaciones.

---

## Paso 3 — Crear el archivo docker-compose.yml

```bash
nano docker-compose.yml
```

Pegar el siguiente contenido y reemplazar los valores marcados:

```yaml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=tudominio.com          # Reemplazar con tu dominio
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://tudominio.com
      - TZ=America/Argentina/Buenos_Aires
      - N8N_ENCRYPTION_KEY=unaClaveAleatoriaMuySegura  # Cambiar por una clave propia
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Guardar y salir: `Ctrl + X` → `Y` → `Enter`

> **Nota:** `N8N_BASIC_AUTH_ACTIVE` fue deprecada en n8n v1+. La autenticación se gestiona desde la propia interfaz al registrar la cuenta "owner" en el primer acceso.

---

## Paso 4 — Iniciar el contenedor

```bash
docker compose up -d
```

Verificar que el contenedor esté corriendo:

```bash
docker compose ps
```

Acceder en el navegador (sin SSL por ahora):

```
http://tu_ip_vps:5678
```

> Si aparece una advertencia de cookie segura, es normal. Se resuelve al configurar SSL en el Paso 5.

---

## Paso 5 — Configurar SSL con NGINX y Certbot

### Instalar NGINX y Certbot

```bash
sudo apt install nginx certbot -y
sudo systemctl stop nginx
```

### Obtener certificado SSL

```bash
sudo certbot certonly --standalone -d tudominio.com
```

Se solicitará un email y aceptar los términos. El certificado se guarda en `/etc/letsencrypt/live/tudominio.com/`.

### Configurar el proxy inverso

```bash
sudo nano /etc/nginx/sites-available/n8n
```

Pegar la siguiente configuración (reemplazar `tudominio.com`):

```nginx
server {
    listen 443 ssl;
    server_name tudominio.com;

    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Soporte para WebSockets (requerido por n8n)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name tudominio.com;
    return 301 https://$host$request_uri;
}
```

### Activar y recargar NGINX

```bash
sudo ln -sf /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/n8n
sudo nginx -t
sudo systemctl start nginx
sudo systemctl reload nginx
```

---

## Paso 6 — Acceder a n8n

Navegar a `https://tudominio.com` y registrar la cuenta "owner" con las credenciales que se deseen. Esta es la cuenta de administrador principal.

---

## Paso 7 — Variables de entorno útiles

Editar `docker-compose.yml` para agregar o modificar según necesidad:

```yaml
environment:
  # Base de datos PostgreSQL (recomendado para producción)
  - DB_TYPE=postgresdb
  - DB_POSTGRESDB_HOST=localhost
  - DB_POSTGRESDB_PORT=5432
  - DB_POSTGRESDB_DATABASE=n8n
  - DB_POSTGRESDB_USER=n8n_user
  - DB_POSTGRESDB_PASSWORD=contraseña_segura

  # Zona horaria
  - TZ=America/Argentina/Buenos_Aires

  # Webhook público
  - WEBHOOK_URL=https://tudominio.com

  # Clave de cifrado de credenciales
  - N8N_ENCRYPTION_KEY=claveAleatoriaMuyLarga
```

Aplicar los cambios:

```bash
docker compose down
docker compose up -d
```

> Para solo reiniciar sin cambios: `docker compose restart`

---

## Paso 8 — Actualizar n8n

```bash
# Descargar la última imagen
docker pull n8nio/n8n

# Reiniciar el contenedor con la nueva versión
docker compose up -d
```

Docker reemplaza el contenedor automáticamente. Los datos en `n8n_data` se mantienen intactos.

---

## Buenas prácticas

| Práctica | Detalle |
|---|---|
| **Volúmenes** | Siempre montar `./n8n_data:/home/node/.n8n` para persistir datos |
| **Backups** | Copiar la carpeta `n8n_data` periódicamente |
| **SSL** | No exponer n8n sin HTTPS en producción |
| **Firewall** | Bloquear el puerto 5678 externamente; solo exponer 80 y 443 |
| **Encryption key** | Definir `N8N_ENCRYPTION_KEY` antes del primer arranque |
| **Recursos** | Limitar CPU/RAM en el compose para evitar sobrecargar el VPS |

### Limitar recursos (opcional)

```yaml
services:
  n8n:
    # ... resto de la config
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
```

### Backup manual del contenedor

```bash
docker cp n8n:/home/node/.n8n ./backup-n8n-$(date +%Y%m%d)
```

---

## Comandos de referencia rápida

```bash
# Ver logs en tiempo real
docker compose logs -f n8n

# Detener n8n
docker compose down

# Reiniciar n8n
docker compose restart

# Ver estado del contenedor
docker compose ps

# Actualizar a la última versión
docker pull n8nio/n8n && docker compose up -d
```

---

## Referencias

- [Documentación oficial de n8n](https://docs.n8n.io/)
- [Variables de entorno de n8n](https://docs.n8n.io/hosting/environment-variables/)
- [Docker Hub — n8nio/n8n](https://hub.docker.com/r/n8nio/n8n)
- [Certbot — Let's Encrypt](https://certbot.eff.org/)
